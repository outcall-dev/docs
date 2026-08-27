# Dashboard (S010)

Outcall ships an operator dashboard that gives you a live view of the
bridge, networks, agent containers, active rules, and pending rule requests
without memorizing CLI commands.

The dashboard is served by the daemon over the host Unix socket as a
self-contained single-page app. It is **host-operator-only** — the agent
socket never serves it, so agent containers cannot reach it even by mistake.

## Accessing the dashboard

The host API binds a Unix domain socket at `/tmp/outcall/host.sock`, not a
TCP port. Browsers can't open Unix sockets directly, so the CLI ships a
built-in TCP-to-Unix bridge.

### Option 1 — `outcall ui` (recommended)

```bash
outcall ui
```

Binds `127.0.0.1:8080`, validates browser requests, forwards them to the daemon
socket, prints a fragment-token URL, and opens your default browser. Press
Ctrl-C to stop. Pass `--port 9000` to use a different loopback port, or
`--no-open` to print the URL without launching a browser.

Do not replace this bridge with `socat`, nginx, or a generic reverse proxy.
Those alternatives omit Outcall's session token, origin checks, request
framing limits, and duplicate-header rejection.

### Direct Unix-socket inspection (Linux)

For headless inspection on Linux without a browser:

```bash
curl --unix-socket /tmp/outcall/host.sock http://_/api/v1/bridge
curl --unix-socket /tmp/outcall/host.sock http://_/api/v1/containers
```

On macOS the socket is container-local. Use `outcall bridge status`,
`outcall ps`, or `outcall ui --no-open`; the CLI transports those requests with
`docker exec` without exposing a host TCP API.

## What the dashboard shows

| View | Backed by | What you see |
|---|---|---|
| **System overview** | `GET /api/v1/bridge`, `/api/v1/dns`, `/api/v1/proxy` | Bridge up/down, nftables active, DNS filter status, proxy stats (active conns, total blocked) |
| **Networks** | `GET /api/v1/networks` | Each `outcall-*` network, its subnet, gateway, and connected containers |
| **Containers** | `GET /api/v1/containers` | All `managed-by=outcalld` agent containers with state, image, network, IP |
| **Rules** | `GET /api/v1/rules` | Currently loaded rules grouped by file, with their CEL conditions |
| **Rule requests** | `GET /api/v1/requests/rules` | Pending rule requests from agents waiting for operator approval |
| **DNS cache** | `GET /api/v1/dns/cache?entries=true` | Cached resolutions with TTL and hit-rate stats |

The dashboard polls these endpoints on a 5-second interval. WebSocket-based
push (S010-FR-010) is on the roadmap; for now polling is good enough on a
single-operator workstation.

## Approving rule requests

When an agent calls `POST /v1/requests/rules` on the agent socket, the
request lands in the queue visible on the dashboard's **Rule requests**
view. Each row has **Approve** and **Reject** buttons that call:

- `POST /api/v1/requests/rules/<id>/approve` — writes a new rule into
  `rules.d/` and triggers a reload, exactly as if you had run
  `outcall rules reload` after editing the file by hand.
- `POST /api/v1/requests/rules/<id>/reject` — drops the request without a
  rule change.

Note: agents submit rule requests to the **agent socket** at
`/v1/requests/rules`. The dashboard approve/reject buttons POST to the
**host API** (operator socket), which is the only surface that can
actually write rule files.

Both actions take effect immediately and are visible to the requesting
agent on its next heartbeat.

## Security

- The daemon API inherits Unix socket file permissions. The `outcall ui`
  loopback bridge adds a random per-process bearer token and rejects non-local
  Host/Origin headers; keep the printed URL private.
- `outcall ui` binds to `127.0.0.1`. Do not replace it with a listener on
  `0.0.0.0` or expose the operator socket through a public reverse proxy.
- The dashboard never proxies traffic; it reads daemon state and posts
  approve/reject decisions. It cannot be used to bypass any rule.

## Platform notes

- **Linux:** supported through the Docker-managed daemon. It receives
  `NET_ADMIN` and `NET_BIND_SERVICE`, plus `CHOWN` and `DAC_OVERRIDE` only for
  native Unix-socket ownership.
- **macOS:** supported through Docker Desktop's Linux runtime. The CLI talks to
  the daemon with `docker exec`; no native macOS netfilter support is required.

## Known limits (v0.1)

- Polling only — no WebSocket push. Slight lag between rule reload and the
  Rules view updating.
- Responsive tables and controls support narrow mobile viewports; wide tables
  scroll within their view instead of widening the page.
- No historical view. Once a rule request is approved/rejected, it leaves
  the queue; check `outcalld` logs for an audit trail.
- No edit-rule UI. Rule files are managed via `rules.d/` + `outcall rules
  reload`. The dashboard is read-and-approve only.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `outcall ui` cannot reach the daemon | Daemon not running | `outcall daemon status`; if stopped, `outcall daemon start` |
| Dashboard loads but tables are empty | No bridge / no containers yet | `outcall bridge up`, then `outcall container create --image …` |
| `404 Not Found` on `/ui/` | Asset path mismatch | Ensure you opened `/ui/` (with trailing slash) — `/ui` 301-redirects on most setups but some shims drop the redirect |
| Dashboard shows stale data | 5-second poll cycle | Refresh once; if still stale check the daemon socket is responding via `curl --unix-socket` |
