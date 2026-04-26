# Outcall Container Guide

How to set up a container to work with Outcall, the protection layers in place, and how to test them manually.

---

## What Outcall Protects Against

Outcall sits between agent containers and the outside world. It enforces policy on:

| Layer | Mechanism | Protects Against |
|-------|-----------|----------------|
| **Network (L3/L4)** | nftables bridge rules | Direct IP connections — TCP/UDP/ICMP to unauthorized IPs |
| **DNS resolution** | DNS filter on bridge IP | Resolving hostnames that policy blocks |
| **HTTP/HTTPS (L7)** | Stateful proxy + SNI inspection | HTTP requests and HTTPS hostname enumeration |
| **Tool execution** | Agent shim + outcalld permission check | Agent running blocked tools or commands |
| **Shell commands** | Agent shim permission check | Agent executing unauthorized shell commands |
| **File access** | Agent shim permission check | Agent reading/writing blocked paths |

**Fail-closed**: if `outcalld` is unreachable, the agent shim exits with code 5 — no action is permitted without a verdict.

---

## Protection Layers

```
Container
├── outcall-agent (shim, read-only mount at /usr/local/bin/outcall)
│   └── Intercepts: tool calls, network requests, shell commands, file access
│       └── Calls outcalld via /run/outcall/agent.sock for verdicts
│
├── /etc/resolv.conf → 10.200.0.1:53 (DNS filter)
│   └── DNS queries evaluated against rule engine → allow or NXDOMAIN
│
├── $HTTP_PROXY / $HTTPS_PROXY → 10.200.0.1:8080 (HTTP proxy)
│   └── HTTP(S) requests inspected via SNI → allow or 403
│
└── Network namespace on outcall bridge (e.g. outcall0, 10.200.0.x/24)
    └── All outbound traffic passes through bridge → nftables drops unauthorized
```

### Layer 1 — nftables Bridge (L3/L4 blocking)

The bridge interface (`outcall0`) runs on the host. All container traffic exits through it. Outcall installs a default-drop nftables ruleset that blocks all outbound connections unless explicitly allowed by a dynamic rule.

- **What it blocks**: raw IP packets (TCP, UDP, ICMP) to any destination not covered by an allow rule
- **How to verify it's blocking**: from inside the container, try `ping 1.1.1.1` or `nc -z 1.1.1.1 443` — both time out or are refused
- **How it allows**: the Dynamic Rules Manager (S009) inserts per-container nftables `accept` rules when a rule request is approved

### Layer 2 — DNS Filter

Containers use the bridge gateway IP as their sole nameserver. Every DNS query is evaluated:

1. Query arrives at `10.200.0.1:53`
2. `outcalld` extracts hostname and record type
3. Rule engine evaluates `dns.query` against active rules
4. **ALLOW** → forwarded to upstream resolver, response returned
5. **BLOCK** → NXDOMAIN returned (agent never learns the real IP)

The DNS filter also maintains a cache keyed on `(hostname, record_type)` with TTL capped by the operator-configured max.

#### Rule-configurable egress mode

DNS allow rules can define follow-up egress behavior:

```yaml
version: "1"
rules:
  - id: allow-dns-ports-ubuntu-com
    condition: 'dns.query == "ports.ubuntu.com"'
    action: allow
    egress:
      mode: direct_ip
      ports: [80, 443]
```

- `egress.mode: proxy` (recommended) keeps access at hostname/SNI policy level and avoids broad IP-level holes.
- `egress.mode: direct_ip` inserts dynamic nftables allows for resolved IPv4 addresses and listed ports.
- If `ports` is omitted in `direct_ip`, Outcall defaults to `[80, 443]`.

- **What it blocks**: resolving hostnames not in the allowlist
- **How to verify**: from inside the container, `nslookup blocked.example.com` returns `NXDOMAIN`
- **What it does NOT block**: Docker's built-in DNS at `127.0.0.11` (local container resolver) — this is local delivery, not forwarded through the bridge

### Layer 3 — HTTP Proxy (L7 inspection)

The HTTP proxy listens on the bridge gateway (`10.200.0.1:8080`). Agent containers are configured with `HTTP_PROXY`/`HTTPS_PROXY` pointing at it.

**HTTP flow:**
1. Agent makes HTTP request (e.g., `GET http://api.github.com/users`)
2. Proxy receives full request — method, path, hostname, headers
3. Populates `http.method`, `http.path`, `http.host`, `http.headers.*` in CEL context
4. Rule engine evaluates; verdict returned
5. **ALLOW** → forward to upstream; **BLOCK** → return `403 Forbidden`

**HTTPS flow:**
1. Agent sends `CONNECT api.github.com:443`
2. Proxy peeks at TLS ClientHello to extract SNI hostname
3. Evaluates `network.hostname` against rules (no decryption — preserves end-to-end TLS)
4. **ALLOW** → tunnel raw bytes; **BLOCK** → return `403`

- **What it blocks**: HTTP requests and HTTPS hostname enumeration by SNI that don't match an allow rule
- **What it does NOT decrypt**: HTTPS payload — end-to-end encryption is preserved

**Best-practice policy model:** prefer proxy-mode hostname rules for internet/CDN domains. `direct_ip` is useful for operational cases like package mirrors, but IP-based access can be broader than one hostname on shared infrastructure.

### Layer 4 — Agent Shim (tool/shell/file permission)

The `outcall-agent` shim is bind-mounted read-only at `/usr/local/bin/outcall`. The agent is configured to route all actions through this shim binary. The shim:

1. On startup: checks in with `outcalld` via `agent.sock` to get a session token
2. On every tool call, shell command, or file access: calls `POST /v1/permissions/check` with the action type and target
3. Waits for verdict (with configurable timeout, default 30s)
4. **ALLOW** → executes the action; **BLOCK** → refuses and returns error to agent
5. **Unreachable** → exits immediately with code 5 (fail closed)

Action types checked:
- `ToolExec` — named tool invocation (`read_file`, `write_file`, etc.)
- `NetworkCall` — outbound network connection
- `FileAccess` — file path read/write
- `ShellExec` — shell command execution

---

## Manual Container Setup

This section describes how to manually create a container that integrates with Outcall, without using the `outcall container create` CLI. Useful for debugging or understanding the requirements.

### Prerequisites

- `outcalld` is running (with the bridge and outcall network already created)
- The following files exist on the host:
  - `/run/outcall/agent.sock` — the agent API socket (created by `outcalld`)
  - `/usr/local/bin/outcall-agent` or equivalent — the agent shim binary (statically linked)
- The outcall bridge is up (`outcall network create` or `outcall bridge up`)

### Step 1 — Create the container on the outcall network

```bash
# Create a container on the outcall network (default: outcall-default)
docker create \
  --name my-agent-container \
  --network outcall-default \
  -it \
  alpine:latest \
  /bin/sh
```

The container must be on `outcall-default` (or whichever network outcall manages).

### Step 2 — Bind-mount the agent socket and shim

```bash
# Inspect the container after creation to verify mounts were applied
# (In production, outcalld applies these automatically via the Docker API)

# For manual testing, use --privileged + nsenter to modify the container
# OR recreate with explicit bind mounts:
docker rm my-agent-container

docker create \
  --name my-agent-container \
  --network outcall-default \
  --mount type=bind,source=/run/outcall/agent.sock,target=/run/outcall/agent.sock \
  --mount type=bind,source=/usr/local/bin/outcall-agent,target=/usr/local/bin/outcall,ro \
  -it \
  alpine:latest \
  /bin/sh
```

### Step 3 — Set environment variables

Inside the container, set the proxy and DNS resolver:

```sh
# DNS — point to the outcall DNS filter on the bridge gateway
# (Replace 10.200.0.1 with your actual bridge gateway IP)
export DNS_RESOLVER="10.200.0.1"

# HTTP proxy — point to the outcall HTTP proxy on the bridge gateway
export HTTP_PROXY="http://10.200.0.1:8080"
export HTTPS_PROXY="http://10.200.0.1:8080"
export NO_PROXY="localhost,127.0.0.1"

# /etc/resolv.conf — override to use outcall DNS
echo "nameserver 10.200.0.1" > /etc/resolv.conf
```

To make these persistent across restarts, use Docker's `--env` flag when creating:

```bash
docker create \
  --name my-agent-container \
  --network outcall-default \
  --mount type=bind,source=/run/outcall/agent.sock,target=/run/outcall/agent.sock \
  --mount type=bind,source=/usr/local/bin/outcall-agent,target=/usr/local/bin/outcall,ro \
  --env HTTP_PROXY="http://10.200.0.1:8080" \
  --env HTTPS_PROXY="http://10.0.0.1:8080" \
  --dns 10.200.0.1 \
  -it \
  alpine:latest \
  /bin/sh
```

### Step 4 — Verify the setup

From inside the container:

```sh
# Verify DNS is pointing to outcall
cat /etc/resolv.conf
# Expected: nameserver 10.200.0.1

# Verify agent socket exists
ls -la /run/outcall/agent.sock
# Expected: srwxr-xr-x 1 root root ... /run/outcall/agent.sock

# Verify shim binary exists
ls -la /usr/local/bin/outcall
# Expected: -r-xr-xr-x ... /usr/local/bin/outcall
```

### Security constraints applied by outcalld (S008-FR-025/026/027)

When `outcalld` creates a container via its Docker Manager, it applies these constraints automatically:

| Constraint | Description |
|------------|-------------|
| Read-only root filesystem | Optional (`--read-only`); prevents container writing to root |
| No privileged mode | Container cannot run in privileged mode |
| Capability dropping | `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, etc. are dropped |
| `--pid 256` | Max PID count limits fork bombs |
| Memory limit (default 512MiB) | Prevents memory exhaustion |
| CPU shares (default 1024) | Fair CPU scheduling |
| Stop timeout (10s) | SIGTERM → 10s → SIGKILL |

The host socket (`/run/outcall/host.sock`) is explicitly denied in the mount validation — any bind mount attempting to include the host socket is rejected before reaching Docker.

---

## Manual Test Steps

### Prerequisites

```bash
# Build the outcall Docker image
make build

# Start outcalld (creates network + bridge + applies nftables)
make start
```

All tests below run from inside a container attached to `outcall-default`.

### Starting an interactive test container

```bash
make start       # ensure daemon is running
make agent       # opens Alpine shell on outcall-default
```

Or manually:

```bash
docker run --rm -it --network outcall-default alpine:latest /bin/sh
```

---

### Test 1 — Network blocking (nftables)

**What it proves**: L3/L4 traffic to unauthorized IPs is blocked by nftables.

```sh
# From the agent container:

# TCP to an unauthorized IP — should time out
nc -z -w 3 1.1.1.1 443
echo "Exit code: $?"          # Expected: non-zero (timeout/refused)

# ICMP ping — should show no reply
ping -c 1 -W 2 1.1.1.1
echo "Exit code: $?"          # Expected: non-zero

# Any TCP to an IP not covered by a dynamic allow rule
curl -s --connect-timeout 3 http://93.184.216.34
# Expected: timeout
```

**Pass**: `nc` and `ping` fail — nftables is dropping the packets.

---

### Test 2 — DNS blocking

**What it proves**: DNS queries to unresolvable or policy-blocked hostnames return NXDOMAIN.

```sh
# DNS lookup that should be blocked — returns NXDOMAIN
nslookup this-host-does-not-exist-12345.example.com 10.200.0.1
# Expected: NXDOMAIN response

# DNS lookup for an allowed hostname (e.g., if github.com is allowed)
nslookup api.github.com 10.200.0.1
# Expected: resolves to an IP address

# Query an external DNS server directly — should fail ( forwarded UDP is blocked by nftables)
nslookup -timeout=2 google.com 8.8.8.8
# Expected: fail (nftables drops the forwarded UDP packet)
```

**Pass**: blocked hostname returns `NXDOMAIN`; direct external DNS fails; allowed hostname resolves.

---

### Test 3 — HTTP proxy blocking

**What it proves**: HTTP requests to disallowed hostnames via the proxy are rejected with 403.

```sh
# First verify the proxy is intercepting
nc -z -w 2 10.200.0.1 8080
echo "Proxy reachable: $?"    # Expected: 0

# HTTP request to a blocked hostname — should get 403
wget -qO- -T 3 --proxy off http://blocked-site.example.com 2>&1
# Expected: empty or connection reset (blocked before reaching the target)

# HTTP request to an allowed hostname (e.g., if example.com is allowed)
wget -qO- -T 3 --proxy off http://example.com
# Expected: HTTP response (allowed)
```

---

### Test 4 — Agent shim check-in

**What it proves**: The agent shim can check in with `outcalld` and receive a session token.

```sh
# From inside the container with the agent socket mounted:

# Check if outcall binary is accessible
/outcall or /usr/local/bin/outcall version 2>/dev/null || echo "shim not directly callable"

# Manually POST to the agent socket (requires netcat or similar tool):
# Replace with your actual check-in call if you have curl/netcat available:
echo '{"action_type": "network_call", "target": "example.com:443"}' | \
  nc -U -w 2 /run/outcall/agent.sock
# Expected: JSON response with verdict

# Check that socket is owned by root and readable by all
ls -la /run/outcall/agent.sock
# Expected: srwxr-xr-x root root ...
```

---

### Test 5 — Full allow-then-block flow

**What it proves**: nftables rules control access, not routing — proving rules can be toggled dynamically.

```bash
# From the host terminal:

# Start with rules up — traffic should be blocked
make test

# Drop the nftables rules
make exec CMD="outcall bridge down"

# Now the agent container can reach the internet (rules are down)
docker exec outcall-daemon ping -c 1 -W 2 1.1.1.1
# Expected: reply (rules down, traffic flows freely)

# Re-apply the rules
make exec CMD="outcall bridge up"

# Agent container is blocked again
docker exec outcall-daemon ping -c 1 -W 2 1.1.1.1
# Expected: no reply (rules up, nftables drops)
```

**Pass**: with rules down, internet is reachable; with rules up, it is not. This proves nftables controls access, not routing tables.

---

### Test 6 — Verify host socket is never mounted

**What it proves**: The host API socket is never accessible from within a container (critical security invariant).

```sh
# Inside a container:
ls /run/outcall/host.sock 2>/dev/null && echo "FAIL: host.sock exists!" || echo "PASS: host.sock not present"

# Try to read from it even if it existed
cat /run/outcall/host.sock 2>/dev/null && echo "FAIL: host.sock readable!" || echo "PASS: host.sock not readable"
```

**Pass**: `host.sock` is not present in the container filesystem.

---

### Test 7 — Daemon unreachable = fail closed

**What it proves**: If `outcalld` stops, the agent cannot execute any action.

```bash
# From the host terminal — stop the daemon
make stop

# Try to run any network call or tool from inside the container
# (If using the shim): it should exit with code 5 immediately
docker exec outcall-daemon /usr/local/bin/outcall 2>&1
# Expected: exit code 5

# Restart the daemon
make start

# Agent should work again
make test
```

**Pass**: daemon stop → shim exits code 5; daemon start → shim recovers.

---

## Container Requirements Summary

| Requirement | Value | Purpose |
|------------|-------|---------|
| Network | `outcall-default` (or managed network) | Route through bridge |
| Agent socket mount | `/run/outcall/agent.sock` (host path) → `/run/outcall/agent.sock` (container) | Agent API communication |
| Shim mount | `/usr/local/bin/outcall-agent` (host) → `/usr/local/bin/outcall` (container, read-only) | Policy enforcement |
| DNS resolver | `10.200.0.1:53` (bridge gateway) | DNS filtering |
| HTTP proxy | `HTTP_PROXY=http://10.200.0.1:8080` / `HTTPS_PROXY=http://10.200.0.1:8080` | L7 HTTP/HTTPS inspection |
| Host socket | Must NOT be mounted | S008-FR-008/009 — host API protection |

---

## Quick Reference

```bash
# Start the full stack
make build && make start

# Run all E2E tests
make test-e2e

# Interactive shell
make agent

# Check bridge status
make exec CMD="outcall bridge status"

# Toggle nftables rules (unlock/lock internet)
make exec CMD="outcall bridge down"   # unlock
make exec CMD="outcall bridge up"      # lock

# Full teardown
make stop
```
