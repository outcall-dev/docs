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
