# Outcall threat model

This document names what Outcall protects against, what it does not, and
the trust boundaries an operator should rely on. It is written from the
perspective of an operator asking *"what can I actually trust here?"*

## The scenario Outcall is built for

You want to run an AI agent inside a container — say, a coding agent that
reads bug reports from Sentry, opens pull requests against a few GitHub
repos, and pulls dependencies from your distro's package mirror. You want
the agent to do exactly that, and nothing else. In particular: no scanning
random hosts, no exfiltrating secrets to a pastebin, no force-pushing to
`main`, no reaching internal services that happen to be on the same
network.

Outcall is the thing that enforces "exactly that, and nothing else" at the
network layer of the host running the container.

## Trust boundaries

Three trust zones. The boundary is what flows between them, and what
Outcall checks at each crossing.

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│   Host (UNTRUSTED FROM AGENT'S POV, TRUSTED FROM OPERATOR'S POV)  │
│                                                                    │
│   ┌────────────────────────────────┐    ┌──────────────────────┐  │
│   │ outcalld (root, CAP_NET_ADMIN) │◄───┤ outcall CLI (root)   │  │
│   │ - rule engine                  │    └──────────────────────┘  │
│   │ - HTTP proxy (L7)              │                              │
│   │ - DNS filter                   │                              │
│   │ - nftables on outcall0 bridge  │                              │
│   │ - agent.sock (SO_PEERCRED)     │                              │
│   └──────────────┬─────────────────┘                              │
│                  │ outcall0 (Linux bridge, nftables filter)        │
│                  │                                                 │
└──────────────────┼─────────────────────────────────────────────────┘
                   │
            ┌──────┴──────┐
            │ veth pairs  │
            │             │
   ┌────────▼────────┐  ┌─▼────────────────┐
   │ Agent container │  │ Agent container  │   <-- UNTRUSTED
   │ (cap_drop ALL,  │  │ (cap_drop ALL,   │       The agent process
   │  outcall-agent  │  │  outcall-agent   │       MUST be treated as
   │  shim, your code)│  │  shim, your code)│       potentially hostile.
   └─────────────────┘  └──────────────────┘
```

| Boundary | What crosses it | What Outcall does at the boundary |
|---|---|---|
| Agent process ↔ container kernel | syscalls | Out of scope. Use Docker `cap_drop`, seccomp profile (M-4), and userns. |
| Container netns ↔ host netns | packets via veth + bridge | nftables on the bridge. Default-drop, explicit allows from the rule engine. |
| Agent shim ↔ daemon | HTTP over `/run/outcall/agent.sock` | SO_PEERCRED resolves PID → container; container is the identity. Rate-limited. Body capped at 64 KiB. |
| Agent ↔ external network (HTTP) | TCP/HTTP via 10.200.0.1:8080 | L7 proxy. Rule engine evaluates pre-CONNECT-200 and post-SNI. 403 on block. |
| Agent ↔ external network (DNS) | UDP/TCP to 10.200.0.1:53 | DNS filter. Rule engine evaluates `dns.query`. NXDOMAIN on block. |
| Operator ↔ daemon | Unix socket `/run/outcall/host.sock` | Root-only by filesystem permissions. |

## What Outcall protects against

Threats Outcall is *intended* to block, and that the implementation
actually blocks (verified by `scripts/e2e/tests/` and the audit linked
below):

| # | Threat | Mitigation |
|---|---|---|
| T-1 | Agent connects to an arbitrary host on the public internet | DNS filter denies the lookup unless a rule allows it; nftables drops direct-IP TCP unless `direct_ip` egress is configured. |
| T-2 | Agent connects to a different agent's container | Forward chain drops bridge-to-bridge traffic unless explicitly allowed; documented as default behavior. |
| T-3 | Agent connects to a host service on a private RFC1918 address | DNS rebinding check warns (H-2 — will block in next pass); nftables drops direct hits. |
| T-4 | Agent in container A claims to be agent B in API calls | Identity is derived from SO_PEERCRED, not from any agent-supplied header or env var. Agents cannot self-identify. |
| T-5 | Agent modifies its own iptables/nftables to grant itself access | Verified: host enforcement wins. See `14-security-boundary.sh`. |
| T-6 | Agent sniffs another agent's traffic on the bridge | veth + bridge isolates broadcast domains per Docker network. No `tcpdump`-style leakage to peers without explicit hairpin config. |
| T-7 | Agent connects to a "good" host but sends data to an evil host via Host-header smuggling | (HTTPS) SNI mismatch re-evaluation closes the tunnel. (HTTP) M-2 will reject mismatched Host headers. |
| T-8 | Agent overwhelms the daemon | Rate limits on agent API; semaphore-bounded proxy accepts; capped DNS cache; 50ms rule-eval budget warns. |
| T-9 | Agent loads its own DNS resolver (DoH/DoT) | Bridge drops all egress except via the L7 proxy / configured direct_ip allows. The agent cannot reach 8.8.8.8:443 unless an allow rule exists, and the cleartext path to a DoT resolver on :853 is also blocked. |
| T-10 | Operator misconfigures rules so a typo silently fails | C-1 / H-1 (audit) will surface CEL errors as `warn!` log entries with the rule id; dead rules are visible in the daemon log. |

## What Outcall does NOT protect against

This is just as important. Out of scope:

| # | Out-of-scope threat | Why / what to use instead |
|---|---|---|
| N-1 | Agent code reads secrets out of the container filesystem and forwards them through an *allowed* destination | Outcall enforces *where* the agent can talk, not *what* it sends. If you allow `api.github.com`, the agent can post anything there. Pair with deliberate secret minimization and least-privilege tokens. |
| N-2 | Agent escapes the container (kernel exploit) | Kernel hardening, userns, seccomp, gVisor/Kata are upstream. |
| N-3 | Agent reads CPU side-channels (Spectre/Meltdown) to leak data | Out of scope. Mitigate at hypervisor / kernel level. |
| N-4 | Operator's machine is compromised | Anything with `CAP_NET_ADMIN` can rewrite the rules. Outcall does not protect against a hostile root user. |
| N-5 | Agent attacks the rule engine itself (e.g. crafted CEL via rule-request endpoint) | Agent-submitted rules are queued for operator approval — they never become live without a human accepting them. |
| N-6 | Agent makes HTTPS to an allowed host with method/path/body the operator did not intend | Without TLS interception (intentionally not implemented), `http.method` / `http.path` / `http.body` are not visible inside the encrypted tunnel. Rule scope is at `http.host` (SNI). |
| N-7 | Supply chain attack on a crate Outcall depends on | `cargo audit` runs in CI; we publish a Cargo.lock with the release. Verify the binary against the signed release (I-2). |
| N-8 | Agent uses an allowed destination as a covert channel (timing, DNS queries, request-volume encoding) | Detecting covert channels is not Outcall's job. If you need this, layer a monitoring tool that watches request rates and timings. |
| N-9 | Operator runs outcalld with `--no-proxy` and expects HTTPS-aware rules to enforce | H-3: this will be a startup error in the next release. Until then, don't do this. |
| N-10 | Agent uses an outbound port that the rule engine evaluates as "allowed by hostname" but the protocol is not HTTP/HTTPS (e.g. `CONNECT example.com:25`) | M-3: tightening this is queued. Today, write your rules to restrict ports explicitly: `dns.query == "x.example" && network.port == 443`. |

## Attacker model

We design for the following attacker:

- The attacker has full code execution inside an agent container.
- The container was started by the operator with `cap_drop ALL` and is
  attached to an outcall-managed network.
- The attacker can read and write any file the container's user can.
- The attacker can issue any syscall not restricted by Docker's seccomp
  default (M-4: a tighter profile is queued).
- The attacker **does not** have root on the host, kernel exploits, or
  the operator's CLI credentials.
- The attacker may have controlled an external service the agent talks to
  (so DNS responses, TLS server certs, and HTTP responses can all be
  adversarial).

Given this model, **Outcall's job is to make it impossible for the agent
to talk to anything not on the explicit allow-list, and to make it visible
when the agent tries.**

## What "default deny" actually means in this codebase

The phrase appears in marketing copy, README, and rules.md. To pin down
its meaning in 2026-05-14:

1. **Rule engine no-match → block.** `engine.rs:48-53` initializes the
   result to `Decision::Block` and only changes it when a matching rule
   has `action: allow`. This is correct.
2. **Rule engine runtime error → no match.** A CEL exception in a rule
   condition means "this rule didn't match" — evaluation continues to
   the next rule. If no rule matches, the default block applies. Correct
   in behavior, but the operator-visibility gap is tracked as H-1.
3. **Rule engine timeout → block.** The agent API wraps evaluation in
   `tokio::time::timeout` and returns `allowed: false` on expiry
   (`agent_api/mod.rs:376`). Correct.
4. **nftables chain policy → currently `accept` with explicit drops.**
   This is the C-3 finding. The deepest enforcement layer should fail
   closed; this is being tightened to `policy drop` with a positive
   allow for non-outcall traffic.
5. **L7 proxy → block on bind failure.** Daemon refuses to start if the
   proxy port can't bind, unless `--no-proxy` is set (correct behavior
   from MAR-257, April).
6. **DNS filter → NXDOMAIN on block, on upstream failure → SERVFAIL.**
   Correct.

## Recommended deployment posture

For the "Sentry → GitHub PR agent" scenario:

1. Run the agent in an outcall-managed container with `cap_drop ALL`.
2. Provide it credentials via filesystem mount, not env vars (env vars
   end up in process listings).
3. Use the example ruleset at `rules.d/examples/sentry-github-agent/`.
4. Run with explicit ports: `network.port == 443` in every rule.
5. Pin the agent's binary by sha256, not a tag.
6. Monitor the daemon log for rule match events — rules that never fire are
   usually wrong; rules that fire constantly are usually too broad.
7. Run `scripts/test-bypass.sh` after every rule-set change in staging
   before you promote.

## Known limitations (current release)

The following issues are acknowledged, partially mitigated, and tracked for
resolution in the next release cycle. They are documented here so operators
can make an informed risk decision rather than discovering them in production.

### BYPASS-03a/03b + PAYLOAD-03 — Private-IP DNS passthrough

The daemon DNS resolver currently passes RFC 1918, loopback, and link-local
IP addresses returned by upstream DNS back to agent containers without
filtering them. An attacker who can influence upstream DNS responses (e.g.
via a compromised or malicious authoritative nameserver) can resolve a
controlled hostname to a private IP and route traffic to internal services,
bypassing intent-based egress rules that block private addresses at the
nftables layer.

**Current mitigation:** nftables drops direct-IP connections to RFC 1918
ranges unless an explicit `direct_ip` rule is configured, so the bypass
requires both DNS influence *and* a gap in nftables rules.

**Roadmap:** Drop private IPs (RFC 1918, loopback, link-local) from upstream
A/AAAA answers before returning them to containers, unless an explicit
`allow-private` rule scope applies.

### BYPASS-08 — ARP cache readable inside agent containers

Agent containers can read `/proc/net/arp`, which exposes MAC and IP addresses
of other containers sharing the same bridge segment. This is informational
disclosure of other containers' L2 identities — it is not a firewall escape
and does not grant the agent network access it does not already have.

**Status:** Accepted limitation. This is inherent to a shared L2 bridge
segment. Operators who require L2 isolation between containers should use
separate outcall-managed networks (one per agent group).

### BYPASS-11 — IPv6 FORWARD not blocked

IPv6 traffic forwarded via a specifically-installed route is not intercepted
by the current nftables `FORWARD` chain rules, which are IPv4-only. An agent
with the ability to configure an IPv6 route could reach the wider network
without going through the L7 proxy or DNS filter.

**Status (fix wave 2):** Resolved for routed IPv6. The base ruleset now
includes `meta nfproto ipv6 drop` in the `forward` chain (both `iifname` and
`oifname` directions), an `output_ipv6_block` chain that drops all IPv6
leaving the bridge interface from the host, and an `input_ipv6_block` chain
that drops unsolicited IPv6 arriving on the bridge. Routed IPv6 to public
destinations (e.g. `2606:4700:4700::1111`) is blocked. See E2E test
`18-ipv6-blocked.sh` (tests 1 and 2).

**Accepted limitation — link-local IPv6 multicast (ff02::/16):** Link-local
multicast packets (e.g. `ping6 ff02::1%veth-agent` sent from within the agent
network namespace) are delivered by the kernel's bridge flooding logic at L2,
without traversing the `FORWARD` hook or the host-namespace `output` hook that
the `output_ipv6_block` chain matches on (`oifname "outcall0"`). The packet
exits via the agent-side veth peer (in the agent's network namespace), not via
`outcall0` in the host namespace, so the nft rule never fires.

This is **not considered an egress bypass** for the following reasons:

1. `ff02::/16` is link-local scope — the kernel will not route these packets
   beyond the L2 segment. They are physically confined to the outcall bridge.
2. The only recipients of bridge-flooded multicast are other containers
   attached to the same bridge and the host's bridge interface — not external
   hosts.
3. Blocking it properly would require injecting nft rules into each agent's
   network namespace (matching `oifname "veth-agent"` inside that netns), which
   is architecturally out of scope for the host daemon.

**Operator guidance:** If you require strict isolation between containers on
the same bridge for L2 multicast traffic, use separate outcall-managed
networks (one bridge per agent group). This matches the guidance for BYPASS-08.

### Default bind addresses — `0.0.0.0`

`--dns-listen` defaults to `0.0.0.0` and `--proxy-addr` defaults to
`0.0.0.0:8080`. On hosts with multiple network interfaces, this exposes the
daemon's DNS filter and HTTP proxy on every interface, not just the outcall
bridge.

**Recommendation:** Override both flags with the bridge IP (`10.200.0.1`)
or the loopback address in any multi-NIC or production deployment:

```
outcalld --dns-listen 10.200.0.1 --proxy-addr 10.200.0.1:8080
```

## Reporting a vulnerability

See `SECURITY.md` at the repository root.
