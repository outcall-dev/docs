# Testing Outcall

## Quick reference

```bash
make build       # build the Docker image
make start       # create network + start daemon + apply nftables
make stop        # stop daemon + remove network
make status      # bridge status
make test        # smoke tests (HTTP, ICMP, DNS — all blocked)
make test-e2e    # full E2E test suite (5 tests)
make agent       # interactive Alpine shell (from your terminal)
make logs        # daemon logs
make exec CMD="outcall bridge down"   # run any command in the daemon
make clean       # stop + remove Docker image
```

## First run

```bash
make build       # one-time: builds the outcall-e2e Docker image (~2 min)
make start       # creates network, starts outcalld, shows bridge status
make test        # runs smoke tests against an isolated Alpine container
make stop        # tears everything down
```

## Architecture

All Outcall binaries are Linux-only (netlink, nftables). On macOS, they run
inside Docker. The Makefile handles this transparently:

```
Your Mac
├── make start
│   ├── docker network create outcall-default  (bridge: outcall0)
│   └── docker run outcall-daemon          (--network host, runs outcalld)
│       └── outcalld
│           ├── attaches to outcall0 bridge
│           ├── applies nftables drop-all rules
│           └── listens on /run/outcall/host.sock
│
├── make test / make agent
│   └── docker run alpine --network outcall-default
│       └── all outbound traffic blocked by nftables
│
├── make exec CMD="outcall bridge status"
│   └── docker exec outcall-daemon outcall bridge status
│       └── talks to outcalld via unix socket
```

## Smoke tests (`make test`)

Spins up a disposable Alpine container on the outcall network and verifies
three types of outbound traffic are blocked:

| Test | Command | Expected |
|------|---------|----------|
| HTTP | `wget -qO- -T 3 http://example.com` | Timeout (blocked) |
| ICMP | `ping -c1 -W 2 1.1.1.1` | No reply (blocked) |
| DNS  | `nslookup -timeout=2 google.com 8.8.8.8` | Failure (blocked) |

Note: Docker's built-in DNS (127.0.0.11) still works inside containers because
it's local delivery, not forwarded through the bridge. The test uses an external
DNS server (8.8.8.8) to verify forwarded UDP is blocked.

Requires the daemon to be running (`make start`).

## E2E test suite (`make test-e2e`)

Self-contained: builds the image if needed, runs a privileged container that
sets up the bridge, nftables, and an agent network namespace, then executes
all test scripts in `scripts/e2e/tests/` in order. See
[docs/tests/](tests/) for detailed descriptions of each test.

Does NOT require `make start` — it runs independently.

```bash
make test-e2e
```

The container needs `NET_ADMIN`, `NET_RAW`, `SYS_ADMIN`, and
`net.ipv4.ip_forward=1`.

### Adding a new E2E test

Drop a numbered `.sh` script in `scripts/e2e/tests/`. The entrypoint picks
them up via glob. Each script receives these env vars:

| Variable | Value | Description |
|----------|-------|-------------|
| `BRIDGE` | `outcall0` | Bridge interface name |
| `BRIDGE_IP` | `10.99.0.1` | Bridge IP address |
| `AGENT_NS` | `agent1` | Network namespace name |
| `AGENT_IP` | `10.99.0.2` | Agent IP inside the namespace |
| `TARGET_IP` | (dynamic) | Container's eth0 IP (forwarded target) |

Exit 0 = pass, non-zero = fail.

## Interactive testing (`make agent`)

Opens an Alpine shell on the outcall network. Run from a real terminal (needs
TTY).

```bash
make start
make agent
```

Inside the shell:

```sh
wget -qO- http://example.com     # blocked
ping -c1 1.1.1.1                  # blocked
nslookup google.com               # resolves (Docker DNS) but can't reach the IP
apk update                        # blocked (can't reach repos)
```

From a second terminal, toggle the rules:

```bash
make exec CMD="outcall bridge down"   # drop nftables rules
# now the agent can reach the internet
make exec CMD="outcall bridge up"     # re-apply rules
# agent is locked down again
```

## Rule egress modes (proxy vs direct_ip)

Outcall rules can explicitly choose egress style for DNS allow entries:

- `egress.mode: proxy` (recommended): no L3/L4 nftables hole, enforce by HTTP proxy + SNI.
- `egress.mode: direct_ip`: opens scoped dynamic nftables allows from DNS A-record answers.

Example rule:

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

## CLI reference

### outcalld (daemon)

Runs on Linux only. Manages the bridge, nftables rules, and host API.

```
outcalld                             # start with defaults
outcalld --bridge outcall0           # specify bridge name
outcalld --socket /tmp/out.sock      # custom socket path
```

| Flag | Default | Description |
|------|---------|-------------|
| `--bridge` | `outcall0` | Bridge interface name |
| `--socket` | `/run/outcall/host.sock` | Host API socket path |

**Startup sequence:**
1. Creates (or attaches to) the bridge
2. Applies the base nftables drop-all ruleset
3. Starts the host API on the unix socket
4. Waits for shutdown (Ctrl-C or SIGTERM)

**Shutdown sequence:**
1. Tears down the nftables table
2. Removes the bridge interface
3. Deletes the socket file

### outcall (host CLI)

Talks to `outcalld` over the unix socket. Platform-independent binary but only
useful when `outcalld` is running.

```
outcall bridge status    # show bridge name, state, nftables status
outcall bridge up        # initialize bridge + apply nftables rules
outcall bridge down      # tear down bridge + remove nftables rules
```

| Flag | Default | Description |
|------|---------|-------------|
| `--socket` | `/run/outcall/host.sock` | Path to outcalld socket |

**Output example:**

```
Bridge:    outcall0
Status:    up
Index:     42
nftables:  active
```

Since the binaries run inside Docker, use `make exec` or `docker exec`:

```bash
make exec CMD="outcall bridge status"
# or
docker exec outcall-daemon outcall bridge status
```

## Host API

The daemon exposes a JSON API on the unix socket. The `outcall` CLI is a thin
client over this API.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/bridge` | Bridge status (fresh netlink + nft check) |
| POST | `/api/v1/bridge/up` | Initialize bridge + nftables rules |
| POST | `/api/v1/bridge/down` | Tear down bridge + nftables rules |

All responses use the `ApiResponse<T>` envelope:

```json
{"success": true, "data": { ... }}
{"success": false, "error": "message"}
```

## Requirements

- Docker Desktop (macOS) or Docker Engine (Linux)
- GNU Make
- For direct (non-Docker) testing on Linux: root or `CAP_NET_ADMIN`, plus
  `nftables` and `iproute2` packages
