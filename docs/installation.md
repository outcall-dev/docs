# Installation

Outcall runs its secure daemon in a Linux runtime. On Linux that is the native
Docker runtime; on macOS it is Docker Desktop's Linux VM. The host CLI manages
the daemon and isolated agent containers in either supported environment.

## Fast install

Install the release binaries:

```sh
curl -fsSL https://outcall.dev/install.sh | sh
```

The installer verifies the SHA-256 checksum of every downloaded binary and,
when Docker is available, the matching `outcalld` image archive before loading
it. This avoids a first-run registry pull.

Then, from the root of the project you want to isolate:

```sh
~/.local/bin/outcall run codex
```

The absolute path works before you add `~/.local/bin` to `PATH`. After that,
use `outcall run codex` or `outcall run claude` from the project you want to
isolate.

Use the desired recipe directly:

```sh
outcall run claude
```

It scaffolds `.outcall/`, checks likely auth/config sources, builds the recipe
image, starts `outcall-daemon` if needed, creates the default network, verifies
the recipe entrypoint in a smoke container, and then launches the isolated agent
container.

Claude users on macOS must authenticate the Linux runtime separately because a
host `/login` lives in Keychain. Start `outcall run claude` interactively once
and complete `/login`, or export `CLAUDE_CODE_OAUTH_TOKEN` after running
`claude setup-token` on the host. Unauthenticated batch/detached runs stop
before building.

If the first run stops on a prerequisite, inspect it directly with:

```sh
outcall doctor --fix claude
outcall doctor --fix codex
```

## Requirements

| Requirement | Why |
|---|---|
| Docker ≥ 20.10 with a Linux runtime | Runs the daemon and agent containers |
| Linux kernel ≥ 5.10 in that runtime | nftables, network bridge, netlink |
| Rust toolchain (build only) | Cargo workspace |
| `NET_ADMIN` on the daemon container | Bridge + nftables management |
| `NET_BIND_SERVICE` on the daemon container | DNS listener on port 53 |

The daemon does not need root if it has the capabilities above. In practice,
running it as a Docker container with `--cap-add` and `--network host` is the
recommended path.

### Kernel prerequisite — `br_netfilter`

Threat **T-2 (agent-to-agent isolation)** is enforced by the nftables FORWARD
chain. The chain only sees L2 bridge traffic if the `br_netfilter` kernel
module is loaded and `net.bridge.bridge-nf-call-iptables=1`. Without it,
two containers on the same bridge can reach each other directly at L2 and
T-2 silently fails.

`outcall run` checks both bridge netfilter values inside the effective Linux
runtime and refuses secure unattended mode unless both are `1`. It does not
silently downgrade the isolation claim.

On a native Linux Docker host, load and persist the module before starting the
daemon:

```sh
# One-shot for the current boot
sudo modprobe br_netfilter
sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
sudo sysctl -w net.bridge.bridge-nf-call-ip6tables=1

# Persist across reboots
echo br_netfilter | sudo tee /etc/modules-load.d/outcall.conf
sudo tee /etc/sysctl.d/99-outcall.conf <<'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

Verify after boot:

```sh
lsmod | grep br_netfilter
cat /proc/sys/net/bridge/bridge-nf-call-iptables   # → 1
```

On macOS, do not run `modprobe` on the macOS host. Docker Desktop owns the
Linux kernel; `outcall doctor` and `outcall run` inspect that runtime from the
daemon container. If the values are not enforceable, fix or restart Docker
Desktop rather than bypassing the preflight.

If the Linux runtime cannot enforce bridge netfilter, Outcall does not support
secure unattended mode on that runtime.

## Manual daemon launch

```sh
outcall daemon start
outcall daemon status
```

The image contains `outcalld`, `outcall`, and `outcall-agent` on `PATH`.
For local development you can still build the debug image from source with
`docker build -f Dockerfile.test -t outcall-daemon .`.

The managed launcher applies:

| Runtime control | Purpose |
|---|---|
| `--cap-drop ALL`, then required capabilities only | Least-privilege bridge, DNS, and socket operation |
| Read-only root + bounded `/tmp` tmpfs | Prevent persistent image mutation |
| Host network/PID namespaces | Enforce the bridge and resolve managed peer identity |
| Docker socket, project rules, and state volume | Container management, active policy, persistence |
| `unless-stopped` restart policy | Restore the control plane after Docker restarts |

## Install from source

Linux:

```sh
git clone https://github.com/outcall-dev/outcall.git
cd outcall
cargo build --workspace --release
sudo install -m 0755 target/release/outcalld /usr/local/sbin/outcalld
sudo install -m 0755 target/release/outcall  /usr/local/bin/outcall
sudo install -m 0755 target/release/outcall-agent /usr/local/bin/outcall-agent
```

The supported operational path is still the Docker-managed daemon. A native
`outcalld` binary is included for development and image construction, not as
the default macOS runtime.

## Verify the install

```sh
outcalld --version
outcall  --version
outcall-agent --version
```

Then let the CLI start the daemon container and check the bridge:

```sh
outcall daemon start
outcall daemon status
outcall bridge status
# Bridge:    outcall0
# Status:    up
# Index:     12
# nftables:  active
```

If the CLI cannot reach the daemon, run `outcall daemon logs` and
`outcall doctor`. On Linux, also inspect `/tmp/outcall/host.sock`; on macOS the
CLI reaches the container-local socket through Docker exec.

## Next steps

- [Quickstart](/docs/guides/quickstart) — isolate Claude Code or Codex in one project.
- [Configuration](/docs/guides/configuration) — every daemon flag explained.
- [Writing rules](/docs/guides/rules) — the rule YAML format and matchers.
