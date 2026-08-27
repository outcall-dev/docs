# Outcall Container Guide

Outcall runs agent workloads through an outcalld-managed Docker container. The
normal entrypoint is:

```sh
cd /path/to/project
outcall run codex       # or: outcall run claude
```

Do not substitute a handwritten `docker run` command for unattended workloads.
The daemon-managed path applies the network, DNS, proxy, mount, capability, and
resource controls as one operation.

## Security Boundary

| Surface | Enforcement | Default |
|---|---|---|
| Raw network | Linux bridge + nftables | Block |
| DNS | Outcall DNS service | NXDOMAIN when no rule allows the query |
| HTTP/HTTPS | Forced proxy environment + bridge firewall | 403/block |
| Container privileges | Non-root process, read-only root, all capabilities dropped, no-new-privileges | Deny |
| Project files | Explicit `/workspace` bind mount | Read-write |
| Other host files/tools | Tokenized host broker + rule evaluation | Not exposed |

The bridge is the non-cooperative boundary. A process that ignores proxy
environment variables still cannot open arbitrary direct connections because
the nftables forward policy drops them. Managed container creation is refused
unless both bridge netfilter hooks are actually enabled.

## Managed Container Layout

```text
managed agent container
├── /workspace                     selected project, read-write
├── /workspace/.outcall            policy/config overlay, read-only
├── $HOME=/home/node               project-local recipe home bind mount
├── process user                   invoking non-root host UID:GID
├── read-only root filesystem
├── writable /tmp tmpfs
├── all Linux capabilities dropped
├── HTTP(S)_PROXY                   daemon proxy on the bridge gateway
├── Docker DNS                     Outcall DNS as its configured upstream
└── outcall-default                managed Docker bridge network
```

Provider auth depends on `--auth`:

| Mode | Transfer |
|---|---|
| `auto` | Saved project choice; otherwise environment credentials, portable credential copy, then a project-local home |
| `env-only` | Matching provider variables plus persistent `.outcall/home/<recipe>/` state |
| `copy` | Portable credential copied into persistent `.outcall/home/<recipe>/` state |
| `mount` | Explicit read-write mount of the complete provider directory, not the host home |

Selected-file staging rejects symlinks, files over 16 MiB, more than 10,000
entries, or more than 100 MiB total. `--include-global-config` opts into the
recipe's selected global instructions, settings, agents, commands, and hooks in
auto, copy, and env-only modes; mount mode already includes the complete
provider directory. Review host-only MCP and executable paths first. Histories,
logs, caches, transcripts, and machine-specific state remain excluded.

Outcall never mounts the whole host home directory. Explicit mount mode can
mount `~/.claude` plus `~/.claude.json`, or `~/.codex`, read-write; the CLI lists
those paths and warns that changes persist on the host. The recipe's project
directory is deliberately writable because the coding agent is expected to
edit it. Other files are absent unless the operator deliberately adds a trusted
mount or host resource.

Claude `/login` credentials on macOS live in Keychain and do not cross into the
Linux runtime. A first interactive `outcall run claude` can complete `/login`
inside the container and persist the Linux credential in the project-local
home. Batch and detached runs require that credential, a
`CLAUDE_CODE_OAUTH_TOKEN`, or an Anthropic API credential before build/launch.

The Codex recipe runs `codex --sandbox danger-full-access` **inside** this
boundary. Codex's Linux `bubblewrap` sandbox requires namespace/capability
support that the managed container intentionally does not receive. This lets
Codex use the mounted workspace and container tools while Docker's read-only
root, dropped capabilities, mount allowlist, and Outcall egress policy continue
to enforce isolation from the host and network.

## Network Enforcement

### L3/L4 bridge

The `outcall-default` Docker network uses the `outcall0` Linux bridge. The base
nftables table allows only:

- established/related return traffic;
- the daemon-owned DNS listener;
- the daemon-owned HTTP proxy listener; and
- explicit dynamic direct-IP grants created from allowed DNS responses.

All other IPv4 forwarding is dropped. IPv6 forwarding and on-link IPv6 output
are dropped by default. Traffic between two containers on the bridge is also
blocked; the daemon refuses managed container creation if
`bridge-nf-call-iptables` or `bridge-nf-call-ip6tables` is not `1`.

### DNS

Docker may display `127.0.0.11` in a user-defined network's `resolv.conf`.
Outcall configures its DNS service as Docker's upstream, so hostname policy is
still evaluated by Outcall. Unknown or blocked names return NXDOMAIN. Direct
queries to public DNS servers are blocked by the bridge firewall.

### HTTP and HTTPS

The daemon owns both uppercase and lowercase proxy variables, and rejects
caller attempts to override them or set `NO_PROXY=*`. Plain HTTP is evaluated
using method, path, host, and headers. HTTPS uses CONNECT host/SNI without
decrypting payloads, so HTTPS method/path/body policy is not available.

`egress.mode: proxy` is the recommended rule mode. `direct_ip` creates a
temporary IP/port firewall grant from an allowed DNS response and is broader on
shared/CDN addresses.

## Container Hardening

Every daemon-managed container receives:

| Control | Value |
|---|---|
| Privileged | `false` |
| Process user | invoking non-root host `UID:GID`; daemon fallback `65532:65532` |
| Capabilities | drop `ALL` |
| Privilege escalation | `no-new-privileges:true` |
| Root filesystem | read-only |
| Writable temp space | `/tmp` tmpfs |
| PID limit | 256 |
| Memory | 512 MiB by default |
| CPU shares | 1024 by default |

The operator socket is a denied mount source and is never passed to agent
containers. Built-in recipe containers also omit the legacy agent helper
socket/shim mounts because Claude and Codex do not transparently route their
internal actions through that wrapper.

## Tools And Files

Commands and files already inside the container are governed by Docker's
boundary. Outcall does not pretend to intercept every syscall or every internal
Claude/Codex tool invocation.

Host-native resources outside `/workspace` use `.outcall/host-resources.yaml`:

```yaml
version: "1"
tools:
  - id: browser-helper
    path: ~/bin/narrow-browser-helper
    default_args: []
files:
  - id: notes
    path: ~/Notes/agent-input
    mode: read-only
auth:
  notes: []
```

A declaration does not grant access. Add the matching project rule:

```sh
outcall allow codex tool:browser-helper
outcall allow codex file:notes
```

When the registry is non-empty, `outcall run` starts a loopback/Unix host
broker, injects a random token and endpoint into the container, and reloads the
project rules. Broker requests must satisfy all three checks: declared resource,
valid bearer token, and an allow verdict from outcalld. Tool grants permit
caller-supplied arguments, so expose a narrow wrapper rather than a shell,
package manager, or general-purpose host CLI.

The v1 broker is request/response, not a transparent transport for a
long-running stdio, SSE, or Streamable HTTP MCP server. Prefer installing the
MCP server in the Linux recipe image, where Outcall's DNS and HTTP(S) policy
still applies. When the server must remain on the host, expose a narrow wrapper
that performs one operation and exits; a protocol-aware host MCP gateway is not
implemented in this release.

The optional `outcall-agent` binary is also an explicit wrapper, not a syscall
interceptor. It fails closed with exit code 5 when its agent socket is
unreachable, but only actions invoked through that wrapper receive per-action
verdicts.

## Naming And Parallel Runs

Recipe names default to the project folder plus the first available numeric
suffix:

```text
/projects/foobar -> foobar-1
second concurrent run -> foobar-2
```

Override the name when useful:

```sh
outcall run codex --name foobar-review --detach
outcall ps
outcall inspect foobar-review
outcall attach foobar-review
outcall logs foobar-review --follow
outcall stop foobar-review
```

Top-level `outcall stop` removes the stopped agent so its numeric name can be
reused. Pass `--keep` to retain it for postmortem logs or inspection; low-level
`outcall container stop` remains stop-only.

Detached interactive agents receive a container TTY and open stdin. Detach from
`outcall attach` without stopping the agent with Ctrl+P, then Ctrl+Q.

## Verification

Inspect the effective hardening without entering the container:

```sh
outcall container inspect --name foobar-1
docker inspect foobar-1 --format '{{json .HostConfig}}'
docker inspect foobar-1 --format '{{json .Mounts}}'
docker network inspect outcall-default
```

The Outcall inspect command lists environment variable names with values
redacted. Avoid writing raw Docker environment output into terminals or CI logs.

Expected invariants:

- `Privileged` is false, `ReadonlyRootfs` is true, and `CapDrop` contains
  `ALL`;
- DNS points at the Outcall gateway and proxy variables cannot be overridden;
- only the project, selected auth/home paths, and read-only `.outcall` overlay
  are mounted;
- `/tmp/outcall/host.sock` is absent;
- the container is attached only to an Outcall-managed network.

Test policy from the container:

```sh
# A host absent from the project rules must fail.
docker exec foobar-1 curl -fsS --connect-timeout 5 https://example.com

# Direct IP and external DNS bypass attempts must fail.
docker exec foobar-1 curl -fsS --connect-timeout 5 https://1.1.1.1
docker exec foobar-1 sh -lc 'command -v nslookup >/dev/null && nslookup openai.com 8.8.8.8'
```

Then grant an exact hostname and retry through the proxy:

```sh
outcall allow codex https://example.com
docker exec foobar-1 curl -fsS --connect-timeout 10 https://example.com
```

Use `make test-e2e` in the root repository for the privileged Linux bridge,
DNS, proxy, IPv6, private-IP, and container-isolation regression suite. That
harness intentionally has broader capabilities because it creates nested
network namespaces; those capabilities are not granted to the production
daemon or agent containers.

## Failure Behavior

- If bridge netfilter is not enforceable, `outcall run` and the daemon
  container API refuse to create the workload.
- If no rule matches, DNS/proxy/rule-engine decisions block by default.
- Managed containers are not configured with an automatic restart policy.
  A Docker restart therefore stops an active agent instead of starting its
  command again without session context.
- The daemon uses Docker's `unless-stopped` restart policy. During any gap, the
  preserved strict base policy blocks agent forwarding; DNS, proxy, and broker
  requests fail until the daemon is ready again.
- After Docker Desktop restarts, run `outcall doctor <recipe>` before starting
  another unattended agent to confirm daemon and rule health.

For command details, see [CLI reference](/docs/guides/cli). For CEL fields and
egress modes, see [Writing rules](/docs/guides/rules).
