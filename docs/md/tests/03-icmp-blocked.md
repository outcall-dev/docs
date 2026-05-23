# 03-icmp-blocked

Verifies that outbound ICMP (ping) from the agent namespace is dropped.

## What it does

1. From inside the agent namespace, pings `1.1.1.1` with a single packet
   and a 2-second timeout.
2. The ICMP echo request is forwarded through the bridge and dropped.
3. If any reply comes back, the test fails. If it times out, the test passes.

## How to run

```bash
make test-e2e    # runs all tests including this one
```

## Why it matters

ICMP is sometimes overlooked in firewall rules. This test confirms the
nftables drop rule covers all IP protocols forwarded through the bridge,
not just TCP and UDP.

## Network path

```
agent1 namespace (10.99.0.2)
  → ICMP echo request to 1.1.1.1
  → outcall0 bridge
  → FORWARD chain: iifname "outcall0" drop  ← blocked here
```

## Script

`scripts/e2e/tests/03-icmp-blocked.sh`
