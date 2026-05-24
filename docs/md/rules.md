# Writing rules

Rules are YAML files in `/etc/outcall/rules.d/`. Every file may declare a
list of rules; the daemon concatenates them, sorts by `priority`, and
evaluates in order for each request. **Default action when no rule matches
is `block`.**

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
| `id` | yes | Unique within the active rule set. Used in logs and structured log output. |
| `description` | no | Free-form. Surface this in dashboards. |
| `condition` | yes | A CEL expression. See [bindings](#bindings). |
| `action` | yes | `allow`, `block`, or `enrich`. Default when no rule matches is `block`. |
| `priority` | no | Integer, lower runs first. Default `100`. Use `< 100` for explicit-deny rules that should fire ahead of allows. |
| `log` | no | `true` to emit a structured log entry when this rule matches. Default `false`. |
| `egress` | no | Per-rule egress configuration when `action: allow`. |

## Definitions and `$name` references

`definitions:` lets you name a sub-expression and reuse it across rules.
References use the `$name` syntax — the daemon expands them recursively
before CEL compilation. Use YAML folded scalars (`>-`) for multi-line
definitions; plain literal blocks (`|`) keep newlines which the CEL parser
rejects.

```yaml
version: "1"
definitions:
  is_github_host: >-
    dns.query == "github.com" ||
    dns.query == "api.github.com" ||
    dns.query.endsWith(".githubusercontent.com")
rules:
  - id: allow-dns-github
    condition: '$is_github_host'
    action: allow
```

Circular references and references to undefined names are errors at load
time.

## Bindings

CEL expressions evaluate against a context object whose namespaces depend
on the layer asking for a verdict. Fields not populated by the asking
layer evaluate to their zero value (empty string, 0, empty map).

If a rule references a field that does not exist in any namespace (e.g.
`ip.dst`, which is not a binding), the CEL runtime raises an error,
outcalld logs a `warn!` with the rule id and treats the rule as no-match.
Operationally: a rule that compiles but never fires is almost always a
typo in a field name — check the daemon log for `warn!` entries naming the rule id.

### `network` (raw L3/L4)

| Field | Type | Example |
|---|---|---|
| `network.hostname` | string | `"api.github.com"` (empty when the layer doesn't know it) |
| `network.ip` | string | `"140.82.121.4"` |
| `network.port` | int | `443` |
| `network.protocol` | string | `"tcp"`, `"udp"` |

### `http`

Outcall does **not** decrypt HTTPS — there is no TLS interception, no CA,
no MITM. What the rule engine sees depends on whether the request is
plaintext HTTP or tunnelled HTTPS:

| Field | Plaintext HTTP | HTTPS (CONNECT + SNI) |
|---|---|---|
| `http.host` | Host header | CONNECT host, then SNI from the TLS ClientHello |
| `http.method` | actual method (`GET`, `POST`, …) | always `"CONNECT"` |
| `http.path` | actual path | always `"/"` |
| `http.headers` | map<string,string> of all request headers | only those sent before the tunnel begins |
| `http.body_size` | bytes (currently `0` — body is not buffered) | `0` |

Filtering HTTPS by method or path is **not possible** without TLS
termination, and Outcall does not terminate TLS. To restrict an HTTPS
service, match on `http.host` (and optionally `dns.query` and
`network.port`). To restrict by method or path, the traffic must be
plaintext HTTP — usually inside a controlled internal network.

### `dns`

| Field | Type | Example |
|---|---|---|
| `dns.query` | string | `"api.openai.com"` |
| `dns.record_type` | string | `"A"`, `"AAAA"`, `"CNAME"` |

### `agent`

| Field | Type | Example |
|---|---|---|
| `agent.name` | string | `"my-agent"` (derived from container name; the trailing `-N` replica suffix is stripped, so `my-agent-1` and `my-agent-2` both resolve to `"my-agent"`) |

`agent.name` is populated only when outcalld can identify the calling
container:

- **Proxy path (HTTP/HTTPS via the proxy port):** the daemon resolves the
  TCP peer's source IP to a container via the `managed-by=outcalld` label,
  then strips the `-N` suffix.
- **Agent shim path (`/run/outcall/agent.sock`):** the daemon reads
  `SO_PEERCRED` from the Unix socket to identify the caller.

If either resolution fails (unmanaged container, traffic that doesn't
transit either path, or a request from outside the network), the `agent`
context is unset and `agent.name` evaluates to an empty string (`""`).
Your rule should treat that case explicitly:

```yaml
# Allow only the CI agent (any replica) to fetch from PyPI.
- id: ci-agent-pypi
  condition: 'agent.name == "ci" && http.host == "pypi.org"'
  action: allow
  egress: { mode: proxy }

# Block any request whose agent identity could not be determined.
- id: deny-unidentified
  condition: 'agent.name == ""'
  priority: 999
  action: block
```

### `docker`

Populated only by rule evaluations originating from a container lifecycle
event (image pull, container create). Not populated for in-flight network
traffic.

| Field | Type |
|---|---|
| `docker.image` | string |
| `docker.command` | list<string> |
| `docker.volumes` | list<string> |
| `docker.env_keys` | list<string> (names only, no values) |
| `docker.capabilities` | list<string> |

### `run`

Populated when the agent shim asks for permission to run a tool, exec a
shell command, or access a file (see `outcall-agent`'s `permissions check`
API).

| Field | Type |
|---|---|
| `run.tool` | string |
| `run.args` | list<string> |
| `run.flags` | list<string> |
| `run.cwd` | string |

## Actions

`action` is one of:

| Action | Behavior |
|---|---|
| `allow` | Permit the request. May carry an `egress:` block. |
| `block` | Deny the request. 403 at the proxy, NXDOMAIN at the DNS filter. |
| `enrich` | Reserved for future use. Does not terminate evaluation. |

The default action when no rule matches is `block`. There is no `deny`
action — write `block`.

## Egress modes

```yaml
action: allow
egress:
  mode: proxy            # enforce at L7 via the HTTP proxy
  ports: [443]           # informational; only used by direct_ip
```

| Mode | Behavior |
|---|---|
| `proxy` (default) | Allow only via the HTTP proxy. SNI/Host enforced at L7. **No TLS decryption.** Recommended for almost all rules. |
| `direct_ip` | Insert a per-rule nftables `accept` for each resolved IPv4/IPv6 of a matching DNS query. Use only when the agent uses raw sockets that can't transit the proxy (e.g. `apt` with parallel range fetches). |

`direct_ip` defaults to ports `[80, 443]` when `ports:` is omitted.

`allow_private_ips: true` may be set under `egress:` for rules that
intentionally target internal services. By default, upstream DNS A/AAAA answers
for private, loopback, link-local, ULA, multicast, and IPv4-mapped addresses are
stripped to prevent DNS rebinding. If all address answers are stripped, the DNS
filter returns SERVFAIL to avoid negatively caching a policy decision as a
nonexistent domain.

For `direct_ip` rules that match AAAA records, the host kernel must support IPv6
nftables expressions (`ip6 saddr/daddr`) so the daemon can insert dynamic IPv6
allow rules. If insertion fails, the daemon logs a warning and leaves the
traffic blocked.

There is **no TLS interception mode** in the current release. Body-content
matching requires a CA-issued certificate to terminate TLS — not yet
shipped. If you need that capability, file an issue.

## Examples

### Allow GitHub clone over HTTPS

```yaml
version: "1"
rules:
  - id: allow-github-https
    condition: 'dns.query == "github.com" || http.host == "github.com"'
    action: allow
```

Method matching (`http.method == "GET"`) cannot be enforced through the
HTTPS proxy — the encrypted tunnel hides it. Allow the host, accept that
the agent could in principle make any HTTPS verb to it.

### Allow the npm registry, only over 443

```yaml
version: "1"
rules:
  - id: allow-npm
    condition: |
      http.host == "registry.npmjs.org" &&
      network.port == 443
    action: allow
    egress:
      mode: proxy
```

### Block a specific image from making any network call

```yaml
version: "1"
rules:
  - id: deny-untrusted-image
    condition: 'docker.image.startsWith("ghcr.io/legacy/")'
    priority: 10
    action: block
```

`docker.image` is populated only at container-create time. To block all
runtime traffic from a container with that image, pair this with an
explicit-by-agent-name rule.

### Allow a tightly-scoped path (plaintext HTTP only)

This pattern only works for plaintext HTTP. For an HTTPS API,
`http.method` and `http.path` are sealed inside the TLS tunnel and cannot
be matched.

```yaml
version: "1"
rules:
  - id: allow-internal-metrics
    condition: |
      http.host == "metrics.internal" &&
      http.method == "GET" &&
      http.path.startsWith("/v1/metrics")
    action: allow
```

### Full worked example

See [`rules.d/examples/sentry-github-agent/`](https://github.com/outcall-dev/root/tree/main/rules.d/examples/sentry-github-agent)
in `outcall-dev/root` for a complete ruleset that locks a coding agent to
Sentry + GitHub + apt mirrors + a single LLM provider, with named deny
rules for git-over-HTTPS and private-network egress.

## Authoring workflow

1. Edit a file in `/etc/outcall/rules.d/`.
2. Run `outcall rules reload` to validate and swap the active set atomically.
   The response reports `files_loaded`, `rules_loaded`, and any warnings.
   Validation failures keep the old set active and return the error.
3. Confirm the rule is firing by watching daemon logs for the rule id
   (set `RUST_LOG=outcalld=debug` for verbose match events).

Rule reloads are atomic at the rule-set pointer level. New requests use the
newly loaded rule set after a successful reload; in-flight requests continue
with whichever rule set their handler had already bound.

A rule that compiles but never matches is almost always wrong — the agent
is either bypassing it (DNS instead of HTTP), you've over-scoped the
condition, **or you've referenced a field that doesn't exist in the
binding table above** (which raises a `warn!` in the daemon log with the
rule id and error, and is treated as no-match).

## Pitfalls

- **Wildcards**: there's no `*.openai.com`. Use CEL string predicates:
  `http.host.endsWith(".openai.com") && http.host != "evil.openai.com.attacker"`.
- **Multi-line definitions**: use folded scalars (`>-`), not literal
  blocks (`|`). The latter preserves newlines which the CEL parser
  rejects.
- **`block` is implicit**. You don't need a catch-all `block` rule — the
  default verdict is `block`. Adding one anyway is fine and surfaces in
  counters.
- **Unused definitions** are flagged at reload with a warning but do not
  fail the load.
- **File extension**: `.yaml` and `.yml` files in the rules directory are
  loaded.
- **`--no-proxy`**: startup fails if any loaded allow rule requires
  `egress.mode: proxy`. Direct-IP-only deployments may use `--no-proxy`, but
  host/SNI HTTP and HTTPS policy requires the proxy.

See [Edge cases](/docs/specs/003-rule-engine/edge-cases) for the
exhaustive list of corner-case behaviors.
