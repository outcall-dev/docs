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

## Reporting a vulnerability

See `SECURITY.md` at the repository root.
