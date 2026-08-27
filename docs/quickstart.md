# Quickstart

Five minutes to an isolated Claude Code or Codex container.

> **Linux runtime required.** On Linux, Outcall uses Docker directly. On
> macOS, Docker Desktop provides the Linux runtime for the daemon and agents.

> **Bridge netfilter is mandatory for secure unattended mode.** `outcall run`
> checks it inside the effective Linux runtime and fails closed. On native
> Linux, configure `br_netfilter` on the host. On macOS, Docker Desktop owns
> the Linux kernel; do not run `modprobe` on macOS. See
> [Installation → Kernel prerequisite](installation.md#kernel-prerequisite--br_netfilter).

## Fast path: Claude Code or Codex

If your goal is "put Claude Code or Codex in a default-deny container without
thinking about bridge internals first", start here.

Install the release binaries:

```sh
curl -fsSL https://outcall.dev/install.sh | sh
```

On Linux and macOS, the installer preloads the matching `outcalld` Docker image
when Docker is available, so the first `outcall run <recipe>` does not depend on an
anonymous registry pull.

Start with the default path:

```sh
~/.local/bin/outcall run codex
```

The absolute path works before you update `PATH`; afterwards, use
`outcall run codex` or `outcall run claude`.

What these do:

- They write `.outcall/` scaffolding for the current project, check Docker and
  generated files, inspect auth candidates and project context, build the
  image, ensure the daemon and default network exist, run a smoke container
  with the recipe entrypoint, and then start the isolated agent container.

Recipes never mount your whole home directory. Auto mode reuses a saved project
choice, uses non-empty provider environment credentials when present, and
otherwise copies only the portable credential into
`.outcall/home/<recipe>/`. Pass `--include-global-config` to additionally copy
the bounded recipe allowlist of global instructions/settings after reviewing
host-only MCP and hook commands.

On macOS, Claude's host `/login` is stored in Keychain and is not a portable
Linux credential. Run `outcall run claude` once without agent arguments and
complete `/login` inside the container, or generate a setup token on the host:

```sh
claude setup-token
export CLAUDE_CODE_OAUTH_TOKEN=your-token
outcall run claude -- -p "Say hi"
```

An unattended or detached command with no portable credential fails before an
image build. `--auth mount` is an explicit read-write opt-in for the complete
provider directory when selected copying is insufficient.

If the fast path stops on a prerequisite, inspect it directly:

```sh
outcall doctor --fix claude
outcall doctor --fix codex
```

If you need to split the flow up, `outcall run <recipe>` expands to:

```sh
outcall init <recipe>
outcall doctor <recipe>
outcall recipe test <recipe>
outcall run <recipe>
```

The intermediate shortcut is `outcall setup <recipe>`. It verifies the same
project scaffold and recipe image without launching the agent.

`outcall doctor <recipe>` now checks the usual first-run failures directly:
Linux host support, Docker daemon availability, `/tmp/outcall`, and the
`br_netfilter` sysctls, plus recipe auth/context candidates.

## Manual path

If you want to understand or operate Outcall below the recipe layer on a Linux
host, use the manual operator flow below. The recipe flow is the supported
cross-platform path.

## 1. Start the daemon

```sh
outcall daemon start
```

Verify the bridge is up and nftables rules are active:

```sh
$ outcall bridge status
Bridge:    outcall0
Status:    up
Index:     12
nftables:  active
```

## 2. Drop in a rule

Default-block is implicit — write only the things the agent may do:

```sh
sudo mkdir -p /etc/outcall/rules.d
sudo tee /etc/outcall/rules.d/agent.yaml > /dev/null <<'EOF'
version: "1"
rules:
  - id: allow-openai
    description: "agent may call the OpenAI API"
    condition: 'http.host == "api.openai.com"'
    action: allow
    egress:
      mode: proxy
EOF
```

Reload the rules:

```sh
outcall rules reload
```

Or, equivalently, POST to the host API directly:

```sh
sudo curl -fsS --unix-socket /tmp/outcall/host.sock \
  -X POST http://localhost/api/v1/rules/reload | jq .
```

The response shows `files_loaded`, `rules_loaded`, and any warnings. If a
rule fails validation, the old set stays active and the error is in the
response body.

## 3. Create the agent network

```sh
$ outcall network create --name agent-net
Network "outcall-agent-net" created (10.200.0.0/24).
```

Names are *suffixes*: `--name agent-net` produces a Docker network
called `outcall-agent-net`. The gateway is the `.1` of the chosen `/24`
(here, `10.200.0.1`) — that's also the address of the DNS filter and HTTP
proxy.

## 4. Run the agent

```sh
docker run -it --rm \
  --network outcall-agent-net \
  --dns 10.200.0.1 \
  -e HTTP_PROXY=http://10.200.0.1:8080 \
  -e HTTPS_PROXY=http://10.200.0.1:8080 \
  -v /tmp/outcall/agent.sock:/run/outcall/agent.sock \
  -v /usr/local/bin/outcall-agent:/usr/local/bin/outcall-agent:ro \
  python:3.12 \
  bash
```

(For the all-in-one path, `outcall container create --image python:3.12
--network agent-net` will wire up DNS, proxy, and shim mounts for you.)

Inside the container, prove enforcement at every layer:

```sh
# Allowed
curl -s -o /dev/null -w "%{http_code}\n" https://api.openai.com/v1/models
# 401 (the request reached OpenAI, OpenAI rejected the empty auth)

# Blocked at DNS
curl -s -o /dev/null -w "%{http_code}\n" https://example.com
# curl: Could not resolve host: example.com  (DNS filter returned NXDOMAIN)

# Blocked at L7 (when DNS happens to resolve from cache)
curl --resolve example.com:443:93.184.216.34 -s -o /dev/null -w "%{http_code}\n" \
     https://example.com
# 403  (HTTP proxy rejected the SNI before opening the upstream tunnel)

# Blocked at L3/L4 (no proxy in the way — direct IP traffic)
nc -zv 1.1.1.1 443
# nc: connect to 1.1.1.1 port 443 (tcp) failed: Connection timed out
```

## 5. Inspect what happened

```sh
outcall dns cache --entries           # which hostnames the filter has seen
outcall network status --name agent-net
outcall bridge status                 # nftables active, bridge up
outcall proxy status                  # active connections, totals, blocks
```

To inspect rules, query the host API:

```sh
curl --unix-socket /tmp/outcall/host.sock http://localhost/api/v1/rules | jq .
```

## What just happened

| Step | Layer | Effect |
|---|---|---|
| 1 | host | `outcalld` brought up the bridge and a default-block nftables table. |
| 2 | rules | Your YAML compiled into a CEL expression and a verdict. |
| 3 | docker | A network attached to the bridge; gateway hosts DNS and proxy. |
| 4 | container | DNS, HTTP proxy, and agent shim wired up. Default-deny is in force. |
| 5 | egress | Each blocked request was rejected at the *highest* layer that saw it. |

## Where to go next

- [Writing rules](/docs/guides/rules) — every matcher, every action.
- [Configuration](/docs/guides/configuration) — every daemon flag.
- [CLI reference](/docs/guides/cli) — every `outcall` subcommand.
- [Troubleshooting](/docs/guides/troubleshooting) — diagnosing the most common failures.
