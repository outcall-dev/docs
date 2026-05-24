# 01-tcp-blocked

Verifies that outbound TCP connections from the agent namespace are dropped by
the nftables FORWARD chain rules.

## What it does

1. From inside the agent namespace, attempts a TCP connection to `1.1.1.1:80`
   (Cloudflare) using `nc` with a 2-second timeout.
2. The packet path is: agent (veth) → outcall0 bridge → kernel routing → eth0.
   At the FORWARD chain, `iifname "outcall0" drop` matches and drops the packet.
3. If the connection succeeds, the test fails. If it times out or is refused,
   the test passes.

## How to run

```bash
# As part of the full suite
make test-e2e

# Individually inside the test container
docker run --rm \
    --cap-add NET_ADMIN --cap-add NET_RAW --cap-add SYS_ADMIN \
    --sysctl net.ipv4.ip_forward=1 \
    --entrypoint bash \
    outcall-e2e -c '
        /entrypoint.sh &  # sets up bridge + namespace
        sleep 3
        source /etc/outcall-test-env 2>/dev/null  # if available
        BRIDGE=outcall0 AGENT_NS=agent1 AGENT_IP=10.99.0.2 \
            bash /tests/01-tcp-blocked.sh
    '
```

## Why it matters

This is the most fundamental test. If TCP isn't blocked, the nftables rules
aren't working and agent containers can reach arbitrary services.

## Network path

```
agent1 namespace (10.99.0.2)
  → veth-agent
  → veth-host (bridge port)
  → outcall0 bridge
  → FORWARD chain: iifname "outcall0" drop  ← blocked here
```

## Script

`scripts/e2e/tests/01-tcp-blocked.sh`
