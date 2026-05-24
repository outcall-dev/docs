# 05-allow-then-reblock

The critical end-to-end test. Proves that nftables rules are what controls
network access, not just a misconfigured route or dead namespace.

## What it does

Three steps:

### Step 1: Verify blocked (sanity check)

Attempts a TCP connection to `1.1.1.1:80` from the agent namespace. Must fail.
If it succeeds, something is wrong with the base rules and the test aborts.

### Step 2: Insert allow rules, verify traffic flows

Inserts two permissive nftables rules at the top of the FORWARD chain:

```
nft insert rule inet outcall forward iifname "outcall0" accept
nft insert rule inet outcall forward oifname "outcall0" accept
```

Then retries the TCP connection. It must succeed — proving the agent can
reach the internet when explicitly allowed.

### Step 3: Restore base rules, verify re-blocked

Flushes the FORWARD chain and re-applies the original drop-all rules. Waits
1 second for conntrack state to expire. Retries the TCP connection — it must
fail again.

## Why this is the most important test

Tests 01-03 prove traffic is blocked, but they can't distinguish between
"nftables is blocking" and "the network is just broken." This test proves
causation: the same namespace, the same route, the same target — only the
nftables rules change, and traffic follows.

## How to run

```bash
make test-e2e    # runs all tests including this one
```

## Sequence diagram

```
Time  Agent namespace          nftables FORWARD chain
────  ─────────────────────    ──────────────────────
  1   TCP → 1.1.1.1:80        iifname outcall0 DROP     → timeout ✓
  2   (rules inserted)         iifname outcall0 ACCEPT
  3   TCP → 1.1.1.1:80        iifname outcall0 ACCEPT   → connected ✓
  4   (rules restored)         iifname outcall0 DROP
  5   TCP → 1.1.1.1:80        iifname outcall0 DROP     → timeout ✓
```

## Script

`scripts/e2e/tests/05-allow-then-reblock.sh`
