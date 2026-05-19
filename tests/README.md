# E2E Test Reference

Each test runs inside a privileged container with a network namespace
simulating an agent container attached to the outcall bridge.

| # | Test | Type | Proves |
|---|------|------|--------|
| 01 | [tcp-blocked](01-tcp-blocked.md) | Negative | Outbound TCP is dropped |
| 02 | [dns-blocked](02-dns-blocked.md) | Negative | Outbound UDP/DNS is dropped |
| 03 | [icmp-blocked](03-icmp-blocked.md) | Negative | Outbound ICMP is dropped |
| 04 | [host-reachable](04-host-reachable.md) | Positive | Agent can reach the bridge IP (host API) |
| 05 | [allow-then-reblock](05-allow-then-reblock.md) | Causation | nftables rules control access, not routing |
| 06 | dns-allowed-ipv4 | Positive | IPv4 DNS allow rule resolves correctly |
| 07 | dns-allowed-ipv6 | Positive | IPv6 DNS allow rule resolves correctly |
| 08 | http-allowed | Positive | HTTP proxy allows a permitted host |
| 09 | https-allowed | Positive | HTTPS CONNECT is allowed for permitted SNI |
| 10 | egress-proxy | Positive | Egress mode `proxy` enforces at L7 |
| 11 | egress-direct-ip | Positive | Egress mode `direct_ip` inserts nftables allow |
| 12 | private-ip-blocked | Negative | RFC 1918 direct-IP traffic is dropped |
| 13 | port-scan-blocked | Negative | Port scan against the bridge is dropped |
| 14 | security-boundary | Security | Container cannot bypass nftables from inside |
| 15 | trusted-repos | Positive | Allow list for trusted registries works end-to-end |
| 16 | hostname-ip-allowlist | Positive | DNS + nftables allow for a multi-IP hostname |
| 17 | host-cli-restrictions | Security | CLI commands are restricted to host socket |

## Running

```bash
make test-e2e              # run the full suite
make test                  # quick smoke tests (requires make start first)
```

## Adding a test

1. Create `scripts/e2e/tests/NN-name.sh` (numbered, executable)
2. Create `docs/tests/NN-name.md` with the description
3. Update this table

The test script receives env vars (`BRIDGE`, `AGENT_NS`, `AGENT_IP`, etc.)
and must exit 0 on pass, non-zero on fail.
