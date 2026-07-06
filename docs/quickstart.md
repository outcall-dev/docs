# Quickstart

Five minutes to an isolated Claude Code or Codex container.

> **Linux only.** `outcalld` manages a kernel network bridge and applies
> nftables rules — both require Linux. macOS hosts can build the workspace
> and run the CLI, but the daemon will not start.

> **Load `br_netfilter` on the host before starting the daemon**, otherwise
> agent-to-agent isolation (T-2) is silently unenforced. See
> [Installation → Kernel prerequisite](installation.md#kernel-prerequisite--br_netfilter).
> Short form: `sudo modprobe br_netfilter && sudo sysctl -w net.bridge.bridge-nf-call-iptables=1`.

## Fast path: Claude Code or Codex

If your goal is "put Claude Code or Codex in a default-deny container without
thinking about bridge internals first", start here.

Install the release binaries:

```sh
curl -fsSL https://outcall.dev/install.sh | sh
```

On Linux, the installer preloads the matching `outcalld` Docker image when
Docker is available, so the first `outcall start` run does not depend on an
anonymous registry pull.

Start with the default path:

```sh
outcall
outcall start
```

Running bare `outcall` prints the recommended first command for the current
project and host, plus the next few useful commands.

If Outcall cannot infer the provider cleanly, choose one explicitly:

```sh
outcall claude
outcall codex
```

That explicit choice also becomes the saved default recipe for the project, so
later runs can go back to `outcall start`.

What these do:

- They write `.outcall/` scaffolding for the current project, check Docker and
  generated files, inspect auth candidates and project context, build the
  image, ensure the daemon and default network exist, run a smoke container
  with the recipe entrypoint, and then start the isolated agent container.

Recipes intentionally avoid mounting your whole home directory. By default they
auto-select copied provider auth/config paths when recipe files exist, and fall
back to env-only when only environment credentials are present.

If the fast path stops on a prerequisite, inspect it directly:

```sh
outcall doctor claude
outcall doctor codex
```

If you need to split the flow up, `outcall run <recipe>` expands to:

```sh
outcall init <recipe>
outcall doctor <recipe>
outcall recipe test <recipe>
outcall recipe run <recipe>
```

`outcall start` uses the same flow, but only auto-selects a provider when it
finds one clear provider signal. It prefers, in order:

- a saved project default recipe from a previous explicit choice
- project context files such as `CLAUDE.md` or `AGENTS.md`
- Claude-only or Codex-only auth candidates on the host

`outcall claude` and `outcall codex` are just direct aliases for
`outcall run claude` and `outcall run codex`.

The intermediate shortcut is:

```sh
outcall setup
outcall start
```

If you want to pin the provider during setup, use `outcall setup claude` or
`outcall setup codex`.

`outcall doctor <recipe>` now checks the usual first-run failures directly:
Linux host support, Docker daemon availability, `/tmp/outcall`, and the
`br_netfilter` sysctls, plus recipe auth/context candidates.

## Manual path

If you want to understand or operate Outcall below the recipe layer, use the
manual operator flow below.

## 1. Start the daemon

```sh
docker run -d --rm \
  --name outcall-daemon \
  --network host \
  --cap-add NET_ADMIN \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/outcall:/tmp/outcall \
  -v /etc/outcall:/etc/outcall \
  ghcr.io/outcall-dev/outcalld:latest \
  --bridge outcall0
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
