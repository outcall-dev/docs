# Security policy

## Supported versions

Outcall is pre-1.0. Only the latest tagged release on
`Outcall-dev/outcall` receives security updates. There are no LTS
branches yet.

| Version | Supported |
|---|---|
| 0.1.x | Yes |
| < 0.1 | No |

## Reporting a vulnerability

If you believe you have found a security issue in Outcall, please
**do not file a public GitHub issue**. Instead:

1. Email `security@outcall.dev` with:
   - A clear description of the issue and its impact
   - Steps to reproduce (ideally a script or container)
   - The Outcall version (`outcall --version`)
   - Whether you've shared this with anyone else and any disclosure
     timeline you have in mind
2. We will acknowledge receipt within 3 business days.
3. We aim to publish a fix within 30 days for high/critical issues.
4. We will credit you in the release notes unless you ask not to be.

If you do not get a reply within 5 business days, please escalate by
opening a GitHub Security Advisory at
<https://github.com/Outcall-dev/outcall/security/advisories>.

## What's in scope

In scope:

- The daemon (`outcalld`) running on a host.
- The CLI (`outcall`) when run by a legitimate operator (root).
- The agent shim (`outcall-agent`) running inside a container.
- The rule engine and CEL evaluation.
- The HTTP proxy and DNS filter enforcement paths.
- The nftables ruleset applied by the daemon.

Out of scope:

- Bugs requiring an attacker to already have root on the host. We
  assume host-root compromises Outcall and that's a feature of the
  trust model — see [`threat-model.md`](./threat-model.md).
- Kernel exploits used to escape the agent container. Use a kernel
  hardening profile (see `M-4` in the audit).
- Side-channel attacks (timing, cache).
- Issues in upstream crates we depend on, except where Outcall
  misuses them. For those, please report to the upstream first.
- Denial of service against the daemon from a process with root on
  the host.

## Coordinated disclosure

We follow a 90-day default disclosure window. If we can't fix an issue
in 90 days we'll communicate that with you before the window expires
and ask for an extension.

If you've already publicly disclosed before contacting us, that's fine
— let us know so we can prioritize.

## What we ask of reporters

- Please don't run automated scanners against production deployments
  you don't own.
- Please don't exfiltrate data beyond what's needed to prove the bug.
- Please give us a reasonable chance to fix before publishing.

## Auditing history

The most recent internal audit is at
[`docs/security/audit-2026-05-14.md`](./audit-2026-05-14.md). The
threat model is at [`docs/security/threat-model.md`](./threat-model.md).
