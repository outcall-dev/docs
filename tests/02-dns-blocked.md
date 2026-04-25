# 02-dns-blocked

Verifies that outbound DNS queries (UDP port 53) from the agent namespace are
dropped by the nftables FORWARD chain rules.

## What it does

1. From inside the agent namespace, sends a DNS query for `example.com` to
   `8.8.8.8` (Google DNS) using `dig` with a 2-second timeout.
2. The UDP packet is forwarded through the bridge and dropped by the same
   `iifname "outcall0" drop` rule that blocks TCP.
3. If the query returns an ANSWER SECTION, the test fails. If it times out,
   the test passes.

## Why 8.8.8.8 and not the default resolver

Docker injects a local DNS resolver at `127.0.0.11` inside containers on
user-defined networks. That DNS is local delivery (INPUT chain), not forwarded
through the bridge. To test that FORWARDED DNS is blocked, the test explicitly
queries an external DNS server.

## How to run

```bash
make test-e2e    # runs all tests including this one
```

## Network path

```
agent1 namespace (10.99.0.2)
  → UDP to 8.8.8.8:53
  → outcall0 bridge
  → FORWARD chain: iifname "outcall0" drop  ← blocked here
```

## Script

`scripts/e2e/tests/02-dns-blocked.sh`
