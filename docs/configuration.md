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

### TLS interception flags (S011 — not yet implemented)

> **Not yet implemented.** S011 is specified but intentionally deferred — see
> `docs/security/threat-model.md`. The flags below are accepted by the daemon
> CLI (they parse without error) but TLS interception is a no-op in the
> current release — no leaf certificates are minted, no bodies are buffered,
> and `egress.mode: intercept` is rejected at rule reload time. Do not rely
> on these flags for enforcement.

| Flag | Default | Purpose (when S011 ships) |
|---|---|---|
| `--ca-cert <path>` | _unset_ | PEM-encoded root CA certificate the proxy will use to sign per-host leaf certs. |
| `--ca-key <path>` | _unset_ | PEM-encoded root CA private key. |
| `--intercept-leaf-ttl-secs <n>` | `86400` | Validity window of generated leaf certificates. |
| `--intercept-body-cap-bytes <n>` | `1048576` | Maximum bytes the proxy will buffer for `http.body` matching. |

## Capability requirements

| Capability | Why |
|---|---|
| `NET_ADMIN` | Manage the bridge, configure interfaces, install nftables. |
| `--network host` | Daemon must operate in the host network namespace. |
| `/var/run/docker.sock` mount | Manage Docker networks; resolve PIDs to containers. |

The daemon does not require `SYS_ADMIN` for current code paths (verified
against `application/outcalld/src/bridge.rs`); some kernels are stricter
about netlink and may need it. Add `SYS_ADMIN` only if the daemon fails
to bring up the bridge with `EPERM`.

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

Networks and containers **outlive the daemon**: if `outcalld` exits, the
bridge and its nftables table are torn down, but Docker networks remain.
When the daemon restarts, it re-attaches and re-applies the ruleset.
During the gap, traffic on the bridge is unfiltered — design your deploys
around this.

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
