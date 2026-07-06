# Troubleshooting

A grab-bag of failure modes and how to diagnose them. If your symptom isn't
here, capture `outcall bridge status`, daemon logs, and the failing
container's `nslookup` and `curl -v` output before opening an issue.

## Daemon won't start

### `bind: address already in use` on port 53

The DNS filter binds `:53` by default. On most distros, `systemd-resolved`
already owns it.

```sh
sudo systemctl disable --now systemd-resolved
# or run the daemon with:
outcalld --dns-port 5353
```

If you change the DNS port, you also need to point your agent containers at
the new port: `--dns 10.200.0.1#5353` doesn't work for most resolvers, so
prefer disabling resolved.

### `bind: address already in use` on port 8080

The HTTP proxy. Move it with `--proxy-addr 10.200.0.1:18080` and update agent
`HTTP_PROXY` / `HTTPS_PROXY` values. Use `--no-proxy` only with direct-IP-only
rule sets; startup fails if loaded allow rules require `egress.mode: proxy`.

### `Operation not permitted` creating the bridge

You're missing `NET_ADMIN`. In Docker, add `--cap-add NET_ADMIN`. On bare
metal, run as root or set capabilities on the binary:

```sh
sudo setcap cap_net_admin,cap_sys_admin+ep /usr/local/sbin/outcalld
```

## CLI says `cannot connect to outcalld`

```sh
outcall bridge status
# Error: cannot connect to outcalld at /tmp/outcall/host.sock — is it running?
#
# Caused by:
#     No such file or directory (os error 2)
```

Means one of:

1. The daemon isn't running. Check your supervisor.
2. The daemon is running but bound to a different socket. Pass
   `outcall --socket /tmp/outcall.sock bridge status`.
3. The user running the CLI lacks permission to open the socket.

## Container can't resolve anything

If every DNS query inside the agent times out:

```sh
# inside the container
cat /etc/resolv.conf
nslookup api.openai.com
```

Common causes:

- **`resolv.conf` doesn't point at the gateway.** Make sure the container is
  started with `--dns 10.200.0.1` (or whatever your gateway is).
- **The DNS filter isn't listening.** From the host: `sudo ss -lnup | grep :53`.
- **No allow rule matches the query.** Check the daemon log for `warn!` entries
  with the rule id — a rule that compiles but never fires usually has a typo in
  a field name. If NXDOMAIN counts climb in the daemon log while no rule matches,
  your condition probably doesn't match what the resolver actually sends — note
  that PTR queries look very different from A queries.

## Container resolves DNS but HTTPS fails

Symptoms: `curl https://example.com` returns 403 or hangs.

```sh
curl -v https://api.openai.com
```

Look for `Proxy-Authenticate` or `403 Forbidden` from the proxy. Usually:

- The Host header / SNI doesn't match any allow rule.
- The HTTP proxy env vars aren't set in the container shell. Check `env | grep -i proxy`.
- The agent is using HTTP/3 / QUIC — Outcall does not yet inspect QUIC. Disable
  it in your client.

## Agent shim returns exit 5

Exit code 5 from the shim means the daemon was unreachable when the shim asked
for a verdict. The shim **fails closed by design** — it never lets a tool run
on a missing verdict.

Diagnose:

```sh
ls -l /tmp/outcall/agent.sock
docker logs outcall-daemon | tail
```

If the agent socket exists but exit-5 persists, the shim is probably mounted
at a path the agent's PID namespace can't see. Re-mount with
`-v /tmp/outcall/agent.sock:/run/outcall/agent.sock`.

## nftables rule isn't firing

```sh
sudo nft list table inet outcall
curl --unix-socket /tmp/outcall/host.sock http://localhost/api/v1/rules | jq .
```

Things to check:

1. Is the rule in the active set? `GET /api/v1/rules` returns what's loaded.
   A YAML edit alone does nothing until you POST `/api/v1/rules/reload`.
2. Does the rule's `egress.mode` match your traffic? `proxy` rules don't
   produce nftables verdicts; only `direct_ip` does.
3. Is the traffic reaching the bridge at all? `tcpdump -i outcall0` should
   show the packet. If not, the container isn't on the agent network.

## Rules reload rejects a file

Reload is atomic. If any rule fails CEL compilation, the *whole* reload is
rejected and the old set stays active. The response body names the file
and the parser error.

```sh
curl --unix-socket /tmp/outcall/host.sock \
     -X POST http://localhost/api/v1/rules/reload | jq .
# {"ok":false,"error":"/etc/outcall/rules.d/agent.yaml:5: CEL parse error: undefined identifier 'http_host'"}
```

The most common error is using `_` in field names — CEL paths are
`http.host`, not `http_host`.

## Performance: latency spikes on first request

On a fresh daemon, the first DNS query for a new host pays the upstream
resolver round-trip. Subsequent queries hit the cache. If your workload is
latency-sensitive, prewarm the cache:

```sh
docker exec my-agent dig +short api.openai.com
docker exec my-agent dig +short github.com
```

## Where to dig deeper

- [Specs S001 / S003 / S006 / S007](/docs/specs) — exact behaviour at every
  layer.
- [Container guide](/docs/guides/container-guide) — verifying enforcement
  manually with `nc`, `curl`, and `dig`.
- [Configuration](/docs/guides/configuration) — every flag.
