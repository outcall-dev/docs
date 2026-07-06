# CLI reference

The `outcall` binary talks to the daemon over its Unix socket. Output is
plain text. The CLI exposes these top-level commands:

```
outcall <subcommand> [flags]
```

| Subcommand group | Purpose |
|---|---|
| `bridge`    | Inspect or change the bridge state. |
| `dns`       | Query the DNS filter; manage its cache. |
| `proxy`     | Inspect the HTTP proxy. |
| `network`   | Create, list, destroy outcall-managed Docker networks. |
| `container` | Run, inspect, stop, remove agent containers. |
| `agent`     | Boot an AI agent container for the current project. |
| `ca`        | Manage the TLS interception CA (init, status, bundle). |
| `daemon`    | Start, stop, or inspect the outcalld daemon container. |
| `rules`     | Hot-reload rules from disk (`outcall rules reload`). |
| `requests`  | Review, approve, or reject agent-submitted rule requests. |
| `recipe`    | Inspect, test, and run known agent runtime recipes. |
| `start`     | Recommended no-brain entrypoint when only one provider is configured. |
| `claude`    | Recommended first-run alias for `outcall run claude`. |
| `codex`     | Recommended first-run alias for `outcall run codex`. |
| `init`      | Scaffold `.outcall/` for the current project, optionally with a recipe. |
| `doctor`    | Check first-run prerequisites, optionally with recipe-specific detail. |
| `setup`     | Run the first-time recipe path: init, doctor, smoke test. |
| `run`       | Recommended first-time path: setup plus recipe launch. |
| `ui`        | Open the operator dashboard in a browser. |

Global flag:

| Flag | Default | Purpose |
|---|---|---|
| `--socket <path>` | `/tmp/outcall/host.sock` | Daemon socket path. |

## run

```sh
outcall run <claude|codex> [--no-build] [--auth auto|copy|mount|env-only] [--detach]
```

This is the lower-level command behind the recommended first-run aliases
`outcall claude` and `outcall codex`. It performs:

```sh
outcall init <recipe>
outcall doctor <recipe>
outcall recipe test <recipe>
outcall recipe run <recipe>
```

Use `outcall setup <recipe>` if you want the scaffold/check/smoke portion
without launching the long-lived agent container yet.

## claude / codex

```sh
outcall claude [--no-build] [--auth auto|copy|mount|env-only] [--detach]
outcall codex  [--no-build] [--auth auto|copy|mount|env-only] [--detach]
```

These are the recommended first commands for new users. They are direct aliases
for `outcall run claude` and `outcall run codex`.

## start

```sh
outcall start [claude|codex] [--no-build] [--auth auto|copy|mount|env-only] [--detach]
```

This is the simplest generic entrypoint. With an explicit provider, it behaves
like `outcall claude` or `outcall codex`. Without one, Outcall inspects the
usual Claude/Codex auth candidates and auto-selects the provider only when the
host clearly matches one of them.

## setup

```sh
outcall setup <claude|codex> [--no-build] [--auth auto|copy|mount|env-only]
```

This runs the scaffold/check/smoke sequence without launching the long-lived
agent container:

```sh
outcall init <recipe>
outcall doctor <recipe>
outcall recipe test <recipe>
```

Use `outcall run <recipe>` after `setup` passes.

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

The container `--name` is a *suffix* — the daemon prepends
`outcall-agent-`. So `--name analyst` becomes `outcall-agent-analyst`.

`stop` sends SIGTERM, waits `--timeout` seconds (default 10), then SIGKILL.

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

## agent

Boot an AI agent container for the current project (S014).

```sh
outcall agent                          # Boot agent with current folder name
outcall agent "analyze this code"      # Boot and pass command to agent
outcall agent --name my-agent          # Custom agent name
outcall agent --image custom:latest    # Custom Docker image
outcall agent --network outcall-default # Attach to an outcall-managed network
outcall agent --detach                 # Run in background
outcall agent --list                   # List running agents
outcall agent --stop                   # Stop agent (auto-detects name)
outcall agent --logs --follow          # Tail agent logs
outcall agent --init                   # Create .outcall/agent.yaml template
```

The agent mounts the current directory at `/workspace` inside the container.
The default Docker network is `outcall-default`; create it with
`outcall network create` before booting an agent, or pass `--network` for a
different outcall-managed network.
Configure per-project settings in `.outcall/agent.yaml`:

```yaml
image: custom-image:latest
name: my-project-agent
volumes:
  - /host/data:/data
env:
  API_KEY: secret
ports:
  - 3000:3000
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
outcall recipe run claude
outcall recipe run codex "inspect this repo"
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
files exist, and whether likely auth/context candidates are present. For Claude
it looks for `ANTHROPIC_API_KEY`, `~/.claude`, `~/.claude.json`, `CLAUDE.md`,
and `.claude/settings.json`. For Codex it looks for `CODEX_ACCESS_TOKEN`,
`CODEX_API_KEY`, `~/.codex/auth.json`, `~/.codex/config.toml`,
`~/.codex/AGENTS.md`, `AGENTS.md`, and `.codex/config.toml`.

`test` is the first-run smoke check. It initializes missing recipe files,
builds the local image unless `--no-build` is passed, stages provider auth,
ensures the daemon and default network exist, and runs the recipe entrypoint
with `--version` inside a short-lived container. This is the fastest way to
see whether the host, auth, and image are ready before starting the real agent.

Recipes intentionally avoid mounting the whole host home directory. Copy or
mount only the selected auth/config paths the recipe reports.

`run` initializes missing recipe files, builds the local recipe image unless
`--no-build` is passed, stages provider auth/config, ensures the daemon and
default network exist, and starts the agent using the same container boot path
as `outcall agent`.

For the two built-in first-run entrypoints, `outcall claude` is an alias for
`outcall run claude` and `outcall codex` is an alias for `outcall run codex`.

Auth transfer modes:

| Mode | Behavior |
|---|---|
| `--auth auto` | Default. Uses copied provider files when recipe user paths exist; otherwise falls back to env-only. |
| `--auth copy` | Copies selected provider files into `.outcall/auth/<id>/home` and mounts that directory as `/home/node`. |
| `--auth mount` | Mounts selected existing provider files directly from the host home directory. |
| `--auth env-only` | Does not copy or mount files; passes matching auth environment variables only. |

Use `--force-auth-copy` to refresh already-staged files. The generated
`.outcall/.gitignore` excludes `.outcall/auth/`; keep that directory treated as
secret material.

Recommended flow:

```sh
outcall init claude
outcall doctor claude
outcall recipe test claude
outcall recipe run claude
```

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
outcall doctor claude
outcall doctor codex
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
