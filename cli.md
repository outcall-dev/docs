# CLI reference

The `outcall` binary talks to the daemon over its Unix socket. Output is
plain text. The CLI today exposes five subcommand groups that mirror the
daemon's subsystems:

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

> **What's not yet a CLI subcommand:** rule management. Today, rules are
> edited as YAML in `--rules-dir` (default `/etc/outcall/rules.d`), and
> reloaded via the host API (see [Reloading rules](#reloading-rules)
> below). A future `outcall rules` subcommand is on the roadmap.

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

## Reloading rules (no CLI today)

Rules are reloaded by POSTing to the host API. Two ways:

```sh
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

Test a rule before deploying it:

```sh
# Edit /etc/outcall/rules.d/agent.yaml, add a new rule for example.com
sudo curl --unix-socket /run/outcall/host.sock \
  -X POST http://localhost/api/v1/rules/reload | jq .

outcall dns test example.com               # would the engine allow this hostname?
```
