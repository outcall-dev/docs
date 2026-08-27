# Configuration

`outcalld` is configured through command-line flags. There is no config file
on the daemon side — the only persistent state is the rule YAML in the
configured `--rules-dir`.

## Daemon flags

| Flag | Default | Purpose |
|---|---|---|
| `--socket <path>` | `/tmp/outcall/host.sock` | Operator API Unix socket (host CLI + UI). |
| `--bridge <name>` | `outcall0` | Linux bridge interface name. Created if missing. |
| `--rules-dir <path>` | `/etc/outcall/rules.d` | Directory of YAML rule files. |
| `--dns-listen <ip>` | `10.200.0.1` | DNS filter bind address (IP only). |
| `--dns-port <port>` | `53` | DNS filter bind port. |
| `--dns-upstream <list>` | from `/etc/resolv.conf` | Comma-separated upstream resolvers (`IP[:port]`). |
| `--proxy-addr <host:port>` | `10.200.0.1:8080` | HTTP proxy bind address. |
| `--no-proxy` | _off_ | Disable the HTTP proxy entirely. Startup fails if loaded allow rules require `egress.mode: proxy`. |
| `--agent-socket-host-path <path>` | `/tmp/outcall/agent.sock` | Agent API Unix socket (one shared socket per host). |
| `--shim-host-path <path>` | `/usr/local/bin/outcall-agent` | Path to the `outcall-agent` shim binary; bind-mounted into agent containers. |
| `--agent-timeout-secs <n>` | `5` | Server-side rule-evaluation timeout for agent permission checks. |
| `--agent-perm-rate <count/seconds>` | `100/10` | Sliding-window rate limit for permission checks per container. |
| `--agent-rule-rate <count/seconds>` | `10/60` | Sliding-window rate limit for rule submissions per container. |
| `--subnet-block <cidr>` | `10.200.0.0/16` | RFC 1918 supernet for `/24` per-network allocation. |

If port `8080` is bound on the bridge address already, pick a free port with
`--proxy-addr 10.200.0.1:18080`. Containers then need their `HTTP_PROXY` env
vars updated to match. Use `--no-proxy` only for direct-IP-only rule sets.

### TLS interception (S011 — not yet implemented)

The daemon exposes no CA or interception flags in the current release. It does
not mint leaf certificates, decrypt HTTPS, or buffer encrypted request bodies;
`egress.mode: intercept` fails rule validation. `outcall ca init` can generate
future-use CA material, but the daemon cannot load or use it yet. Do not install
that CA into a trust store for current Outcall deployments.

## Capability requirements

| Capability | Why |
|---|---|
| `NET_ADMIN` | Manage the bridge, configure interfaces, install nftables. |
| `NET_BIND_SERVICE` | Bind the managed DNS listener on port 53. |
| `CHOWN`, `DAC_OVERRIDE` | Linux Unix-socket transport only: assign mounted socket ownership to the invoking operator. |
| `--network host` | Daemon must operate in the host network namespace. |
| `--pid host` | Resolve network PIDs to managed Docker container identities. |
| `/var/run/docker.sock` mount | Manage Docker networks; resolve PIDs to containers. |

The daemon does not require `SYS_ADMIN`. Docker-exec transport on macOS omits
`CHOWN` and `DAC_OVERRIDE`; native Unix-socket transport on Linux adds them.
A bridge-netfilter preflight failure must be fixed in the Linux runtime, not
bypassed by broadening the daemon capability set.

For local daemon development, set `OUTCALL_DAEMON_IMAGE=<local-tag>` before
`outcall run`. Recipe-driven rules-mount restarts preserve the currently
selected daemon image; the environment variable explicitly overrides it.

## Logging

The daemon emits structured logs via `tracing-subscriber` to stderr. The
log level is controlled by the `RUST_LOG` environment variable, not a flag:

```sh
RUST_LOG=info outcalld …                       # default
RUST_LOG=outcalld=debug,hyper=warn outcalld …  # per-target
RUST_LOG=trace outcalld …                      # everything
```

Each subsystem logs under a stable target name (`bridge`, `network`,
`rule_engine`, `dns`, `proxy`, `agent_api`, `host_api`,
`docker_manager`). Filter by target when shipping to your log backend.

## State and persistence

| Path | What lives there |
|---|---|
| `--rules-dir` (default `/etc/outcall/rules.d/`) | Rule YAML, edited by operators. |
| `/var/lib/outcall/` | Persisted state (rule requests, dynamic rules). |
| `/tmp/outcall/host.sock` | Operator socket (recreated on each daemon start). |
| `/tmp/outcall/agent.sock` | Agent socket (recreated on each daemon start). |

Networks and containers **outlive the daemon**. During graceful shutdown,
`outcalld` removes dynamic direct-egress grants and preserves the bridge with
its strict base nftables policy, so managed containers remain fail closed.
DNS, proxy, broker, and control-socket services are unavailable until the
daemon restarts; requests that depend on them fail.
On restart, the daemon re-attaches to the bridge, replaces the ruleset, and
rediscovers containers by the `managed-by=outcalld` label. Use the explicit
`outcall bridge down` command only after all managed workloads have stopped.
The CLI creates `outcall-daemon` with Docker's `unless-stopped` restart policy,
so Docker Engine and Docker Desktop restarts bring the control plane back
without a manual command. `outcall daemon stop` removes the container and does
not restart it.

## Daemon lifecycle

`outcalld` initialises in dependency order:

```
1. Bridge          create / attach + base nftables
2. Rule engine     load YAML, compile CEL conditions
3. Network         create default network, assign gateway IP
4. DNS + Proxy     bind on gateway IP (parallel)
5. Docker manager  ready to attach containers
6. Host API        operator socket goes live
7. Agent API       agent socket goes live
```

If any P1 step fails, startup aborts. The full sequence is documented in
[the spec index](/docs/specs).

## Hot reload

Rule changes are picked up by POSTing to the host API:

```sh
curl --unix-socket /tmp/outcall/host.sock \
  -X POST http://localhost/api/v1/rules/reload | jq .
```

The daemon validates the new ruleset, then atomically swaps it in. Failed
validation keeps the old set active and returns the error in the response.

## Health and readiness

There is no dedicated `/api/v1/health` or `/api/v1/ready` endpoint today.
For now, supervisors can probe `/api/v1/bridge` (`bridge_status`) — a
healthy daemon will return JSON with `up: true` and `nftables: active`.
A proper liveness/readiness pair is on the roadmap.
