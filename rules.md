# Writing rules

Rules are YAML files in `/etc/outcall/rules.d/`. Every file may declare a
list of rules; the daemon concatenates them, in filename order, into one
active rule set.

## Anatomy

```yaml
version: "1"
rules:
  - id: allow-openai
    description: "agent may call the OpenAI API only"
    condition: 'dns.query == "api.openai.com" || http.host == "api.openai.com"'
    action: allow
    egress:
      mode: proxy
```

| Field | Required | Purpose |
|---|---|---|
| `version` | yes | YAML schema version. Currently `"1"`. |
| `id` | yes | Unique within the active rule set. Used in logs and `outcall rules show`. |
| `description` | no | Free-form. Surface this in dashboards. |
| `condition` | yes | A CEL expression. See [matchers](#matchers). |
| `action` | yes | `allow` or `block`. Default action when no rule matches is `block`. |
| `egress` | no | Per-rule egress configuration when `action: allow`. |

## Matchers

Conditions are written in [CEL](https://github.com/google/cel-spec). The rule
engine evaluates the condition against a context object whose shape depends on
the layer that asked for a verdict.

### DNS context

| Field | Type | Example |
|---|---|---|
| `dns.query` | string | `"api.openai.com"` |
| `dns.type` | string | `"A"`, `"AAAA"`, `"CNAME"` |

### HTTP context

Outcall does **not** decrypt HTTPS — there is no TLS interception, no CA, no
MITM. What the rule engine sees depends on whether the request is plaintext
HTTP or tunnelled HTTPS:

| Field | Plaintext HTTP | HTTPS (CONNECT + SNI) |
|---|---|---|
| `http.host` | Host header | CONNECT host, then SNI from the TLS ClientHello |
| `http.method` | actual method (`GET`, `POST`, …) | always `"CONNECT"` |
| `http.path` | actual path | always `"/"` |
| `http.scheme` | `"http"` | `"https"` |

Practical consequence: **filtering HTTPS by method or path is not possible**
without TLS termination, and Outcall does not terminate TLS. To restrict an
HTTPS service, match on `http.host` (and optionally `dns.query`) only. To
restrict by method or path, the traffic must be plaintext HTTP — usually
inside a controlled internal network.

### Network context (raw L3/L4)

| Field | Type | Example |
|---|---|---|
| `net.dst_ip` | string | `"140.82.121.4"` |
| `net.dst_port` | int | `443` |
| `net.protocol` | string | `"tcp"`, `"udp"`, `"icmp"` |

### Agent context

| Field | Type | Example |
|---|---|---|
| `agent.name` | string | `"my-agent"` (derived from container name; the trailing `-N` replica suffix is stripped, so `outcall-agent-my-agent-1` and `outcall-agent-my-agent-2` both resolve to `"my-agent"`) |

`agent.name` is populated only when outcalld can identify the calling container:

- **Proxy path (HTTP/HTTPS via the proxy port):** the daemon resolves the
  TCP peer's source IP to a container via the `managed-by=outcalld` label,
  then strips the `-N` suffix.
- **Agent shim path (`/run/outcall/agent.sock`):** the daemon reads
  `SO_PEERCRED` from the Unix socket to identify the caller.

If either resolution fails (unmanaged container, traffic that doesn't transit
either path, or a request from outside the network), `agent` is unset and
referencing `agent.name` evaluates to `null` — your rule should treat that
case explicitly. For container-image matching use `docker.image` instead.

```yaml
# Allow only the CI agent (any replica) to fetch from PyPI.
- id: ci-agent-pypi
  condition: 'agent.name == "ci" && http.host == "pypi.org"'
  action: allow
  egress: { mode: proxy, ports: [443] }

# Block dev replicas from PyPI even though the rule above wouldn't match them.
- id: dev-agents-no-pypi
  condition: 'agent.name == "dev" && http.host == "pypi.org"'
  action: deny

# Belt-and-braces: if agent identity is unknown, deny network egress.
- id: deny-unidentified
  condition: 'agent == null'
  action: deny
  priority: 999
```

You can mix contexts in a single rule. The engine surfaces only the fields
relevant to the layer asking for a verdict — references to absent fields
evaluate to `null` and short-circuit `&&`/`||` correctly.

## Actions

```yaml
action: allow
egress:
  mode: proxy            # enforce at L7 via the HTTP proxy
  ports: [443]           # required when mode is direct_ip
```

| Mode | Behaviour |
|---|---|
| `proxy` (default) | Allow only via the HTTP proxy. SNI/Host enforced at L7. **No TLS decryption.** Recommended for most rules. |
| `direct_ip` | Insert a per-rule nftables `accept` for each resolved IPv4/IPv6 of the matching DNS query. Use only when the agent uses raw sockets that can't transit the proxy. |
| `intercept` | **Optional, off by default.** Terminate TLS at the proxy using an operator-provided CA so the rule engine can match on `http.method`, `http.path`, and (optionally) `http.body`. Requires `--ca-cert` / `--ca-key` on the daemon and the CA installed in the agent's trust store. See [TLS interception](#tls-interception-mode-intercept) below. |

`direct_ip` defaults to ports `[80, 443]` when omitted.

## TLS interception (`mode: intercept`)

For most rules, `mode: proxy` is the right answer — you can match HTTPS by
hostname (CONNECT host + SNI), and the agent's TLS session is preserved
end-to-end.

When you genuinely need to enforce policy on the *contents* of an HTTPS
request — only allow `POST /v1/messages` against `api.anthropic.com`, or
reject a JSON body that contains a forbidden field — you need the proxy to
decrypt. That's what `mode: intercept` gives you, with explicit trade-offs:

- **You provision a CA.** `outcall ca init` produces a fresh root CA. The
  cert is the trust anchor; the key signs per-host leaf certs the proxy
  presents to the agent. The key is sensitive material — store it as you'd
  store any private key.
- **The agent must trust the CA.** Mount the cert into the container's
  trust store (`/usr/local/share/ca-certificates/outcall.crt` on
  Debian/Ubuntu, then `update-ca-certificates`). For Python's `certifi`
  bundle, use `SSL_CERT_FILE`.
- **Pinning breaks.** Hosts that pin certificates at the application layer
  (Google services, some banks) will reject the proxy's leaf cert.
  Document those hosts and use `mode: proxy` for them.
- **mTLS breaks.** A client cert presented by the agent terminates at the
  proxy. mTLS-protected upstreams must use `mode: proxy`.
- **Bodies enter the daemon's memory.** When `match_body: true`, the
  request body up to the configured cap (default 1 MiB) is buffered in
  the proxy. Sensitive bodies should be considered visible to the daemon.

### Setup

```sh
# 1. Generate a CA (once per host).
outcall ca init --out /etc/outcall/ca/

# 2. Restart outcalld with the CA flags.
outcalld \
  --bridge outcall0 \
  --ca-cert /etc/outcall/ca/ca.crt \
  --ca-key  /etc/outcall/ca/ca.key

# 3. Confirm the CA is loaded.
outcall ca status

# 4. Distribute the CA cert to your container build (or mount at runtime).
outcall ca bundle > /etc/outcall/ca/ca.pem
```

### Enabling on a rule

```yaml
version: "1"
rules:
  - id: anthropic-messages-only
    description: "agent may POST messages, nothing else on this host"
    condition: |
      http.host == "api.anthropic.com" &&
      http.method == "POST" &&
      http.path.startsWith("/v1/messages")
    action: allow
    egress:
      mode: intercept
```

A rule with `mode: intercept` is rejected at reload if no CA is loaded —
the daemon refuses the whole rule set rather than silently degrading.

### Inspecting payloads

When you need to match on body contents, opt in per rule:

```yaml
- id: openai-no-system-override
  description: "block prompts that try to override the system role"
  condition: |
    http.host == "api.openai.com" &&
    http.method == "POST" &&
    http.body != null &&
    !http.body.contains('"role":"system"')
  action: allow
  egress:
    mode: intercept
    match_body: true
```

`http.body` is `null` when:
- The rule does not set `match_body: true`.
- The body exceeds `--intercept-body-cap-bytes` (default 1 MiB).
- The body fails UTF-8 decode (lossy replacement is attempted first).

Always test for nullness before string operations: a rule with
`http.body.contains(...)` and a non-text payload would error otherwise.

The full spec — every flag, every error code, every edge case — is in
[S011: TLS Interception](/docs/specs/011-tls-interception).

## Examples

### Allow GitHub clone (HTTPS), nothing else

```yaml
version: "1"
rules:
  - id: allow-github-https
    condition: |
      dns.query == "github.com" ||
      http.host == "github.com"
    action: allow
```

Method matching (`http.method == "GET"`) cannot be enforced through the
HTTPS proxy — the encrypted tunnel hides it. Allow the host, accept that
the agent could in principle make any HTTPS verb to it.

### Allow the npm registry

```yaml
version: "1"
rules:
  - id: allow-npm
    condition: 'http.host == "registry.npmjs.org"'
    action: allow
```

The proxy already enforces HTTPS in practice — DNS only resolves what the
rule set allows, and `registry.npmjs.org` only serves TLS. No `http.scheme`
filter needed.

### Block everything from a specific image

```yaml
version: "1"
rules:
  - id: deny-untrusted-image
    condition: 'agent.image.startsWith("ghcr.io/legacy/")'
    action: block
```

### Allow a tightly-scoped path (plaintext HTTP only)

This pattern only works for plaintext HTTP. For an HTTPS API like
`api.anthropic.com`, method and path are sealed inside the TLS tunnel and
cannot be matched.

```yaml
version: "1"
rules:
  - id: allow-internal-metrics
    condition: |
      http.scheme == "http" &&
      http.host == "metrics.internal" &&
      http.method == "GET" &&
      http.path.startsWith("/v1/metrics")
    action: allow
```

## Authoring workflow

1. Edit a file in `/etc/outcall/rules.d/`.
2. Run `outcall rules reload --dry-run` to validate.
3. Run `outcall rules reload` to swap the active set.
4. Run `outcall rules counters` after a few minutes to confirm the rule is
   actually firing.

A rule that compiles but never matches is almost always wrong — the agent is
either bypassing it (DNS instead of HTTP), or you've over-scoped the
condition.

## Pitfalls

- **Wildcards**: there's no `*.openai.com`. Use CEL string predicates:
  `http.host.endsWith(".openai.com") && http.host != "evil.openai.com.attacker"`.
- **Empty rule files** are rejected — `version: "1"` with no `rules:` is a
  parse error, on the assumption that you meant to write something.
- **`block` is implicit**. You don't need a catch-all `block` rule — the
  default verdict is `block`. Adding one anyway is fine and surfaces in
  counters.

See [Edge cases](/docs/specs/003-rule-engine/edge-cases) for the exhaustive
list of corner-case behaviours.
