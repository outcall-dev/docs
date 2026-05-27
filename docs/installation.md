# Installation

Outcall is a Linux-only daemon. It manages a kernel network bridge and applies
nftables rules — both require Linux. macOS hosts can build the workspace and
run the CLI, but `outcalld` itself will not start outside Linux.

## Requirements

| Requirement | Why |
|---|---|
| Linux kernel ≥ 5.10 | nftables, network bridge, netlink |
| Docker ≥ 20.10 | Container management API |
| `nft` binary | nftables ruleset application |
| Rust toolchain (build only) | Cargo workspace |
| `NET_ADMIN` capability | Bridge + nftables management |

The daemon does not need root if it has the capabilities above. In practice,
running it as a Docker container with `--cap-add` and `--network host` is the
recommended path.

### Kernel prerequisite — `br_netfilter`

Threat **T-2 (agent-to-agent isolation)** is enforced by the nftables FORWARD
chain. The chain only sees L2 bridge traffic if the `br_netfilter` kernel
module is loaded and `net.bridge.bridge-nf-call-iptables=1`. Without it,
two containers on the same bridge can reach each other directly at L2 and
T-2 silently fails.

The daemon attempts to flip the sysctl on startup, but loading the module
itself requires `CAP_SYS_MODULE` — which the recommended container deploy
does not grant. **Load the module on the host before starting the daemon:**

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

If you cannot enable `br_netfilter` (locked-down kernel, hardened host),
treat T-2 as out of scope and put each agent on its own outcall-managed
network — bridge separation gives you isolation without relying on FORWARD.

## Install via Docker (recommended)

A prebuilt image will be published once releases land. For now, build from
source and run it as a container:

```sh
git clone https://github.com/outcall-dev/outcall.git
cd outcall
docker build -f Dockerfile.test -t outcall-e2e .

docker run -d --rm \
  --name outcall-daemon \
  --network host \
  --cap-add NET_ADMIN \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /run/outcall:/run/outcall \
  -v /etc/outcall:/etc/outcall \
  outcall-e2e \
  ./target/debug/outcalld --bridge outcall0
```

`Dockerfile.test` is a `cargo build --workspace` (debug) image and does not put
the binaries on `PATH`, so `outcalld` is invoked by its build path from the
image's `/app` workdir. A release image with `outcalld` on `PATH` ships with
the first published release.

Required mounts:

| Mount | Purpose |
|---|---|
| `/var/run/docker.sock` | Manage Docker networks, look up containers by PID |
| `/run/outcall` | Unix sockets for host CLI and agent shim |
| `/etc/outcall` | Rule files and persisted state |

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

A systemd unit, sysctl knobs, and capability defaults will ship with a future
package release; for now run the daemon under your service manager of choice.

## Verify the install

```sh
outcalld --version
outcall  --version
```

Then start the daemon and check the bridge is up:

```sh
sudo outcalld --bridge outcall0 &
outcall bridge status
# Bridge:    outcall0
# Status:    up
# Index:     12
# nftables:  active
```

If you see `daemon unreachable`, check the socket path
(`/run/outcall/host.sock`) and the daemon's permission to bind it.

## Next steps

- [Quickstart](/docs/guides/quickstart) — bring up an agent container with rules.
- [Configuration](/docs/guides/configuration) — every daemon flag explained.
- [Writing rules](/docs/guides/rules) — the rule YAML format and matchers.
