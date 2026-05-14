# Dashboard (S010)

Outcall ships a read-only operator dashboard that gives you a live view of the
bridge, networks, agent containers, active rules, and pending rule requests
without memorizing CLI commands.

The dashboard is served by the daemon over the host Unix socket as a
self-contained single-page app. It is **host-operator-only** — the agent
socket never serves it, so agent containers cannot reach it even by mistake.

## Accessing the dashboard

The host API binds a Unix domain socket at `/run/outcall/host.sock`, not a
TCP port. Browsers can't open Unix sockets directly, so the CLI ships a
built-in TCP-to-Unix bridge.

### Option 1 — `outcall ui` (recommended)

```bash
outcall ui
```

Binds 127.0.0.1:8080 → the host socket, prints the URL, and opens your
default browser. Press Ctrl-C to stop. Pass `--port 9000` to bind a
different port, or `--no-open` if you don't want the browser launched
automatically.

Equivalent under the hood to running `socat TCP-LISTEN:8080,reuseaddr,fork
UNIX-CONNECT:/run/outcall/host.sock` — handy if you'd rather use socat
directly:

```bash
socat TCP-LISTEN:8080,reuseaddr,fork UNIX-CONNECT:/run/outcall/host.sock
```

### Option 2 — `nginx` (for an always-on workstation)

```nginx
server {
    listen 127.0.0.1:8080;
    location / { proxy_pass http://unix:/run/outcall/host.sock:; }
}
```

### Option 3 — `curl --unix-socket` (no browser needed)

For headless inspection without spinning up a shim:

```bash
curl --unix-socket /run/outcall/host.sock http://_/api/v1/bridge
curl --unix-socket /run/outcall/host.sock http://_/api/v1/containers
```

## What the dashboard shows

| View | Backed by | What you see |
|---|---|---|
| **System overview** | `GET /api/v1/bridge`, `/api/v1/dns`, `/api/v1/proxy` | Bridge up/down, nftables active, DNS filter status, proxy stats (active conns, total blocked) |
| **Networks** | `GET /api/v1/networks` | Each `outcall-*` network, its subnet, gateway, and connected containers |
| **Containers** | `GET /api/v1/containers` | All `managed-by=outcalld` agent containers with state, image, network, IP |
| **Rules** | `GET /api/v1/rules` | Currently loaded rules grouped by file, with their CEL conditions |
| **Rule requests** | `GET /api/v1/rule-requests` | Pending rule requests from agents waiting for operator approval |
| **DNS cache** | `GET /api/v1/dns/cache?entries=true` | Cached resolutions with TTL and hit-rate stats |

The dashboard polls these endpoints on a 5-second interval. WebSocket-based
push (S010-FR-010) is on the roadmap; for now polling is good enough on a
single-operator workstation.

## Approving rule requests

When an agent calls `POST /api/v1/rule-request` from inside its container,
the request lands in the queue visible on the dashboard's **Rule requests**
view. Each row has **Approve** and **Reject** buttons that call:

- `POST /api/v1/rule-requests/<id>/approve` — writes a new rule into
  `rules.d/` and triggers a reload, exactly as if you had run
  `outcall rules reload` after editing the file by hand.
- `POST /api/v1/rule-requests/<id>/reject` — drops the request without a
  rule change.

Both actions take effect immediately and are visible to the requesting
agent on its next heartbeat.

## Security

- The dashboard inherits Unix socket file permissions (`/run/outcall/host.sock`
  is `0660`, owner `root:outcall` by default). If you can read the socket,
  you can use the dashboard — there is no separate auth layer.
- Treat the socat/nginx shim as a privileged endpoint: bind to `127.0.0.1`,
  not `0.0.0.0`, and don't expose port 8080 to the network.
- The dashboard never proxies traffic; it reads daemon state and posts
  approve/reject decisions. It cannot be used to bypass any rule.

## Platform notes

- **Linux:** fully supported. The daemon needs `NET_ADMIN` and `SYS_ADMIN`
  to manage nftables and the bridge, so run it under systemd or via
  `outcall daemon start` (which uses Docker with the right caps).
- **macOS (development only):** outcalld cannot run natively because it
  needs Linux netfilter. Use a Linux VM (lima/colima/UTM) or run it inside
  a privileged Docker container. The dashboard itself works the same once
  the host socket is reachable.

## Known limits (v0.1)

- Polling only — no WebSocket push. Slight lag between rule reload and the
  Rules view updating.
- Desktop layout only. Mobile-responsive design is out of scope for v0.1.
- No historical view. Once a rule request is approved/rejected, it leaves
  the queue; check `outcalld` logs for an audit trail.
- No edit-rule UI. Rule files are managed via `rules.d/` + `outcall rules
  reload`. The dashboard is read-and-approve only.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `curl: (7) Couldn't connect to server` against the socat shim | Daemon not running | `outcall daemon status`; if stopped, `outcall daemon start` |
| Dashboard loads but tables are empty | No bridge / no containers yet | `outcall bridge up`, then `outcall container create --image …` |
| `404 Not Found` on `/ui/` | Asset path mismatch | Ensure you opened `/ui/` (with trailing slash) — `/ui` 301-redirects on most setups but some shims drop the redirect |
| Dashboard shows stale data | 5-second poll cycle | Refresh once; if still stale check the daemon socket is responding via `curl --unix-socket` |
