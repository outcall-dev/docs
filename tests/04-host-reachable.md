# 04-host-reachable

Verifies that the agent CAN reach the bridge IP. This is the **positive** test
— it confirms the host API surface is accessible from inside agent containers.

## What it does

1. A simple HTTP server (socat) runs on `$BRIDGE_IP:8080` (10.99.0.1:8080),
   simulating the host API socket surface.
2. From inside the agent namespace, `curl` fetches `http://10.99.0.1:8080/`.
3. If the request succeeds and returns the expected body, the test passes.
   If it fails, the test fails.

## Why this is correct behavior

Traffic from the agent to the bridge's own IP (10.99.0.1) is LOCAL delivery.
The kernel routes it through the INPUT chain, not the FORWARD chain. The
nftables drop rules only apply to FORWARDED traffic (to external IPs), so
local delivery to the bridge IP is allowed.

This is intentional: in production, the agent socket (`agent.sock`) is exposed
at the bridge/host level. Agents need to reach it to request permissions.

## How to run

```bash
make test-e2e    # runs all tests including this one
```

## Network path

```
agent1 namespace (10.99.0.2)
  → TCP to 10.99.0.1:8080
  → outcall0 bridge (local address)
  → INPUT chain (not FORWARD)  ← allowed, delivered locally
  → socat HTTP server responds
```

## Script

`scripts/e2e/tests/04-host-reachable.sh`
