# CLI reference

The `outcall` binary talks to the daemon over its Unix socket. Output is
plain text. The CLI exposes these top-level commands:

```sh
outcall
outcall <subcommand> [flags]
```

Running bare `outcall` prints the recommended first command for the current
project and host, followed by the shortest useful onboarding commands.

| Subcommand group | Purpose |
|---|---|
| `bridge`    | Inspect or change the bridge state. |
| `dns`       | Query the DNS filter; manage its cache. |
| `proxy`     | Inspect the HTTP proxy. |
| `network`   | Create, list, destroy outcall-managed Docker networks. |
| `container` | Run, inspect, stop, remove agent containers. |
| `ca`        | Prepare or inspect CA state for the future TLS-interception feature. |
| `daemon`    | Start, stop, or inspect the outcalld daemon container. |
| `rules`     | Hot-reload rules from disk (`outcall rules reload`). |
| `requests`  | Review, approve, or reject agent-submitted rule requests. |
| `recipe`    | Inspect, initialize, and test known agent runtime recipes. |
| `init`      | Scaffold `.outcall/` for the current project, optionally with a recipe. |
| `doctor`    | Check first-run prerequisites, optionally with recipe-specific detail. |
| `auth`      | Stage selected provider auth/config without launching an agent. |
| `allow`     | Add a recipe template or exact-host grant to project rule YAML. |
| `policy`    | Explain the effective project-local policy and default-deny behavior. |
| `ps` / `logs` / `attach` / `stop` | Manage named recipe containers without Docker commands. |
| `setup`     | Run the first-time recipe path: init, doctor, smoke test. |
| `run`       | Recommended first-time path: setup plus recipe launch. |
| `ui`        | Open the operator dashboard in a browser. |

Global flag:

| Flag | Default | Purpose |
|---|---|---|
| `--socket <path>` | `/tmp/outcall/host.sock` | Daemon socket path. |

## run

```sh
outcall run <claude|codex> [--name NAME] [--no-build]
            [--auth auto|copy|mount|env-only] [--include-global-config]
            [--detach] [--keep] [-- ARGS...]
```

This is the only recipe agent launch command. On a new project it performs the
same initialization, checks, image build, auth staging, daemon/network setup,
and `--version` smoke test as `outcall setup`, then launches the agent. Once the
project scaffold exists, it preserves that scaffold and launches through the
same daemon-managed container API. Use `--no-build` only when the local recipe
image already exists.

Use `outcall setup [recipe]` if you want the scaffold/check/smoke portion
without launching the long-lived agent container yet.

Default selected-file staging copies only portable credentials. Add
`--include-global-config` to copy the recipe's selected user-level settings and
instructions in auto, copy, or env-only mode after checking that MCP/hook
commands are valid in Linux. Mount mode already includes the complete provider
directory.
Argument-free `--detach` runs receive a container TTY and open stdin; reconnect
with `outcall attach <name>` and detach with Ctrl+P, then Ctrl+Q.

## setup

```sh
outcall setup [claude|codex] [--no-build]
              [--auth auto|copy|mount|env-only] [--include-global-config]
```

This runs the scaffold/check/smoke sequence without launching the long-lived
agent container. Without an explicit provider, it uses the saved
default/project-context/host-auth detection order:

```sh
outcall init <recipe>
outcall doctor <recipe>
outcall recipe test <recipe>
```

Use `outcall run <recipe>` after `setup` passes.
Successful attached and one-shot runs remove their stopped container so the
lowest available `<folder>-N` name can be reused. Pass `--keep` to retain a
completed container for `docker logs` or `docker inspect`; detached containers
remain running until explicitly stopped.

## bridge

```sh
outcall bridge status
outcall bridge up
outcall bridge down
```

`status` reports the bridge name, kernel state (up/down), the bridge index,
and whether nftables rules are active. `up` and `down` are idempotent — running
either twice has no effect.

```
$ outcall bridge status
Bridge:    outcall0
Status:    up
Index:     12
nftables:  active
```

## dns

```sh
outcall dns status                       # listening?  upstream resolvers?
outcall dns test <hostname> [--type A]   # ask the rule engine: would this resolve?
outcall dns cache [--entries]            # cache size; with --entries, list contents
outcall dns flush                        # drop the cache
```

`outcall dns test api.openai.com` is the cleanest way to confirm a rule
intends what you think — it asks the engine without sending any traffic.

## proxy

```sh
outcall proxy status
```

Reports the proxy's listen address, active connections, total requests, and
total blocked. This is the only `proxy` subcommand today.

## network

```sh
outcall network create  [--name <suffix>] [--subnet <cidr>] [--gateway <ip>]
outcall network status  [--name <suffix>]
outcall network list
outcall network destroy [--name <suffix>]
```

If `--name` is omitted, all of these target the default network
(`outcall-default`). The name is a *suffix* — the daemon prepends `outcall-`,
so `--name my-agents` becomes the Docker network `outcall-my-agents`.

`create` allocates a `/24` from the daemon's `--subnet-block` if `--subnet`
is omitted. Gateway defaults to the `.1` of the chosen subnet.

`destroy` refuses if any containers are still attached — stop or remove the
containers first.

## Agent container lifecycle

```sh
outcall ps
outcall inspect <name>
outcall logs <name> [--follow]
outcall attach <name>
outcall stop <name> [--keep]
```

These top-level commands are the normal recipe-agent lifecycle. `inspect`
shows environment names with every value replaced by `<redacted>`. `attach`
resolves the daemon-managed container identity before invoking Docker, so an
unmanaged container with a reused name cannot be selected. `stop` removes the
stopped agent by default so its numeric name can be reused; pass `--keep` to
retain it for logs or inspection. Low-level `container stop` never removes.

## container

```sh
outcall container create  --image <image> [--network <suffix>] [--name <suffix>]
                          [--memory <e.g. 256m>] [--cpu-shares <n>]
outcall container list
outcall container inspect --name <name>
outcall container stop    --name <name> [--timeout <secs>]
outcall container remove  --name <name> [--force]
outcall container pull    --image <image>
```

An explicit container `--name` is used exactly. If omitted, this low-level API
generates an `outcall-<8-hex>` name; recipe runs use `<folder>-N` names instead.

`stop` sends SIGTERM, waits `--timeout` seconds (default 10), then SIGKILL.
`inspect` returns environment names with every value replaced by `<redacted>`.

## Reloading rules

Rules are reloaded via the CLI or the host API:

```sh
# Using the CLI (recommended)
outcall rules reload

# Using curl over the unix socket
curl --unix-socket /tmp/outcall/host.sock -X POST http://localhost/api/v1/rules/reload

# In a script:
sudo curl -fsS --unix-socket /tmp/outcall/host.sock \
  -X POST http://localhost/api/v1/rules/reload \
  | jq .
```

The response includes the number of files loaded, the number of rules
compiled, and any warnings. If validation fails, the previous rule set
remains active and the response includes the error.

Listing currently loaded rules:

```sh
curl --unix-socket /tmp/outcall/host.sock http://localhost/api/v1/rules | jq .
```

## Rule requests

Agents may submit proposed rule files through the agent API. Those rules are
queued for operator review and never become active until approved.

```sh
outcall requests list
outcall requests approve rr-aabbcc112233
outcall requests reject rr-aabbcc112233 --reason "too broad"
```

`approve` writes the submitted rule file through the host API and reloads the
active rule set atomically. `reject` records the reason so the agent can poll
the request status and report it to the operator.

## Logging

Daemon log level is controlled by the `RUST_LOG` environment variable, not a
flag. Examples:

```sh
RUST_LOG=info  outcalld …                 # default
RUST_LOG=outcalld=debug,hyper=warn outcalld …
RUST_LOG=trace outcalld …                  # everything, very loud
```

Logs go to stderr in `tracing-subscriber`'s text format.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Success. |
| `1` | Any error, including a daemon that is unreachable (socket missing, permission denied) or any failed operation. |
| `2` | Bad arguments (clap). |

## Examples

Bring up a fresh agent network and a Python container attached to it:

```sh
outcall bridge up
outcall network create --name python-agents
outcall container create \
    --image python:3.12-slim \
    --network python-agents \
    --name analyst \
    --memory 1g
outcall container list
```

## recipe

Initialize and run a known agent runtime profile.

```sh
outcall recipe list
outcall recipe show claude
outcall init claude
outcall init codex --force
outcall recipe doctor claude
outcall recipe test claude
outcall run claude
outcall run codex -- "inspect this repo"
```

Built-in recipes:

| Recipe | Purpose |
|---|---|
| `claude` | Claude Code image scaffold, Anthropic API egress rules, and Claude context/auth transfer notes. |
| `codex` | Codex CLI image scaffold, OpenAI/ChatGPT egress rules, and Codex context/auth transfer notes. |

`init` writes:

```text
.outcall/recipes/<id>/recipe.yaml
.outcall/recipes/<id>/Dockerfile
.outcall/recipes/<id>/README.md
.outcall/recipes/<id>/context.md
.outcall/rules/<id>.yaml
.outcall/agent.yaml
.outcall/.gitignore
```

`doctor` checks whether Docker and Git are available, whether generated recipe
files exist, and whether portable credentials, optional user configuration, and
project context are present. For Claude it checks the supported environment
variables and `~/.claude/.credentials.json` separately from selected settings,
hooks, `CLAUDE.md`, and `.claude/settings.json`. For Codex it checks
`CODEX_ACCESS_TOKEN`, `CODEX_API_KEY`, `~/.codex/auth.json`, user configuration,
`AGENTS.md`, and `.codex/config.toml`.

`test` is the first-run smoke check. It initializes missing recipe files,
builds the local image unless `--no-build` is passed, stages provider auth,
ensures the daemon and default network exist, and runs the recipe entrypoint
with `--version` inside a short-lived container. This is the fastest way to
see whether the host, image, and managed runtime are ready before starting the
real agent. It reports missing portable auth as a warning because `--version`
does not require a credential.

Recipes intentionally avoid mounting the whole host home directory. Copy or
mount only the selected auth/config paths the recipe reports.

`run` initializes missing recipe files, builds the local recipe image unless
`--no-build` is passed, stages provider auth/config, ensures the daemon and
default network exist, and starts the agent through outcalld's managed
container API.

For the built-in first-run entrypoints, use `outcall run claude` or
`outcall run codex`.

Auth transfer modes:

| Mode | Behavior |
|---|---|
| `--auth auto` | Default. Reuses a saved project choice; otherwise uses non-empty environment credentials, a portable credential, then an isolated project home. |
| `--auth copy` | Copies portable credential files into `.outcall/home/<id>` and mounts it at the validated Linux home (`/home/node`) inside the container. |
| `--auth mount` | Explicitly mounts the complete provider directory read-write (`~/.claude` plus `~/.claude.json`, or `~/.codex`). |
| `--auth env-only` | Passes matching auth environment variables and reuses persistent `.outcall/home/<id>` state without copying host credentials. `--include-global-config` may still copy the bounded global-config allowlist. |

Selected-file staging never follows symlinks and rejects files over 16 MiB,
more than 10,000 entries, or more than 100 MiB total. Add
`--include-global-config` to copy the recipe's selected global
settings/instructions, and use `--force-auth-copy` to refresh selected host
files. The generated `.outcall/.gitignore` excludes
`.outcall/home/` and `.outcall/auth/`; treat both as secret runtime state.

On macOS, Claude's host `/login` credential is in Keychain and cannot be copied
into Linux. Run `outcall run claude` interactively once to persist a Linux login
in the project home, or export `CLAUDE_CODE_OAUTH_TOKEN` from
`claude setup-token`. Batch and detached commands fail before setup/build when
no portable credential exists; `--version`, `--help`, and provider login
commands remain available.

Recommended flow:

```sh
outcall init claude
outcall doctor --fix claude
outcall recipe test claude
outcall run claude
```

## host-broker

The host broker exposes only resources declared in
`.outcall/host-resources.yaml` and still asks outcalld for a rule verdict on
every request. `outcall run` starts the correct transport automatically when
the registry is non-empty. The manual commands are primarily for debugging:

```sh
outcall host-broker serve
outcall host-broker serve-tcp --listen 127.0.0.1:17890
```

The Unix transport is used on Linux. Docker Desktop uses a random loopback TCP
port reachable as `host.docker.internal`, protected by a random bearer token
and an exact generated transport rule. The TCP command refuses non-loopback
listeners. A host tool grant allows caller-supplied arguments, so declare a
narrow wrapper instead of a shell or other general-purpose binary.

`/v1/tool/exec` is a bounded one-shot command API. It does not transparently
forward long-lived stdio, SSE, or Streamable HTTP MCP sessions. Install MCP
servers in the Linux recipe image when possible, or expose a narrow host wrapper
that performs one operation and exits.

## init

Scaffold the current project for Outcall use.

```sh
outcall init
outcall init claude
outcall init codex --force
```

`outcall init` creates:

```text
.outcall/agent.yaml
.outcall/rules/
.outcall/.gitignore
```

`outcall init <recipe>` adds the recipe scaffold on top of that base layout.
By default `init` refuses to overwrite generated files; pass `--force` when
you intentionally want to refresh them.

## doctor

Check first-run prerequisites without talking to the daemon API.

```sh
outcall doctor
outcall doctor --fix claude
outcall doctor --fix codex
```

The top-level `doctor` checks the local scaffold plus command availability.
It also checks Linux host support, Docker daemon reachability, the default
socket directory (`/tmp/outcall`), and the `br_netfilter` sysctls that gate
agent-to-agent isolation. The recipe-specific form adds auth candidate checks
and project context checks.

Test a rule before deploying it:

```sh
# Edit /etc/outcall/rules.d/agent.yaml, add a new rule for example.com
sudo curl --unix-socket /tmp/outcall/host.sock \
  -X POST http://localhost/api/v1/rules/reload | jq .

outcall dns test example.com               # would the engine allow this hostname?
```
