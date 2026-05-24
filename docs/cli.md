# CLI reference

The `outcall` binary talks to the daemon over its Unix socket. Output is
plain text. The CLI exposes eleven subcommand groups:

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
| `ui`        | Open the operator dashboard in a browser. |

Global flag:

| Flag | Default | Purpose |
|---|---|---|
| `--socket <path>` | `/run/outcall/host.sock` | Daemon socket path. |

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
curl --unix-socket /run/outcall/host.sock -X POST http://localhost/api/v1/rules/reload

# In a script:
sudo curl -fsS --unix-socket /run/outcall/host.sock \
  -X POST http://localhost/api/v1/rules/reload \
  | jq .
```

The response includes the number of files loaded, the number of rules
compiled, and any warnings. If validation fails, the previous rule set
remains active and the response includes the error.

Listing currently loaded rules:

```sh
curl --unix-socket /run/outcall/host.sock http://localhost/api/v1/rules | jq .
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
| `1` | Generic error. |
| `2` | Bad arguments (clap). |
| `5` | Daemon unreachable (socket missing, permission denied). |

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
outcall agent --detach                 # Run in background
outcall agent --list                   # List running agents
outcall agent --stop                   # Stop agent (auto-detects name)
outcall agent --logs --follow          # Tail agent logs
outcall agent --init                   # Create .outcall/agent.yaml template
```

The agent mounts the current directory at `/workspace` inside the container.
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

Test a rule before deploying it:

```sh
# Edit /etc/outcall/rules.d/agent.yaml, add a new rule for example.com
sudo curl --unix-socket /run/outcall/host.sock \
  -X POST http://localhost/api/v1/rules/reload | jq .

outcall dns test example.com               # would the engine allow this hostname?
```
