# Testing

Outcall has three layers of automated tests:

| Layer | Where | When you run it |
|---|---|---|
| **Unit tests** | `#[cfg(test)] mod tests` blocks inline in `outcalld/src/**/*.rs` | every `cargo test` |
| **Integration tests** | `outcalld/tests/*.rs` (real syscalls, root + Linux required) | every `cargo test` if you have the caps |
| **End-to-end harness** | `Makefile` + `scripts/e2e/tests/*.sh` (Docker-based) | `make test` / `make test-e2e` |

This guide walks each layer in order. The first two are what most people
mean by "tests"; the third is a cross-binary smoke harness used to confirm
the whole stack lights up on a fresh Docker host.

## Unit tests (`cargo test`)

Run the whole workspace's tests from `application/`:

```sh
cd application
cargo test --workspace --all-targets
```

Today there are **69 unit tests** across the workspace. They cover (most-
to-least populous):

| File | Tests | What's covered |
|---|---|---|
| `outcalld/src/rules/engine.rs` | 16 | CEL evaluation, reload, rule priority, dynamic merge |
| `outcalld/src/proxy/mod.rs` | 12 | SNI extraction, host:port parsing, request line parsing, CRLF detection |
| `outcalld/src/network/mod.rs` | 11 | Subnet allocation, CIDR validation |
| `outcalld/src/agent_api/mod.rs` | 7 | Agent permission-check protocol |
| `outcalld/src/docker/mod.rs` | 7 | Docker network create/destroy paths |
| `outcalld/src/dynamic/mod.rs` | 5 | Dynamic rule merge into the active set |
| `outcall-agent/src/main.rs` | 4 | Tool-invocation parsing (bash, fetch, file_read) |
| `outcalld/src/dns/mod.rs` | 3 | DNS filter happy path + cache |
| `outcall-ui/src/lib.rs` | 2 | UI types |
| `outcalld/src/rules/model.rs` | 1 | Rule YAML deserialization (incl. `egress.mode: direct_ip`) |

Unit tests are pure-Rust. They run on macOS, Linux, and CI without any
capabilities, sockets, or Docker.

### Running a subset

```sh
cargo test -p outcalld                     # just the daemon
cargo test -p outcalld rules::             # just the rule engine module
cargo test -p outcalld sni_empty           # one named test
cargo test -- --nocapture                  # show println!/dbg! output
cargo test -- --test-threads=1             # serial execution (handy when state leaks)
```

### Async tests

Anything that needs the Tokio runtime uses `#[tokio::test]`:

```rust
#[tokio::test]
async fn reload_picks_up_new_rules() { … }
```

You don't have to set up a runtime yourself.

## Integration tests (`cargo test --test ...`)

Integration tests live in `outcalld/tests/*.rs` — separate files, compiled
against the public crate API. They exercise real syscalls.

Today there is **one** integration test:

- `outcalld/tests/bridge_integration.rs` — creates and destroys the
  `outcall0` bridge, applies and tears down nftables rules. Verifies state
  via `ip link show` and `nft list table`.

It needs Linux and `CAP_NET_ADMIN` (or root):

```sh
sudo cargo test -p outcalld --test bridge_integration -- --nocapture
```

On macOS the test is gated behind `#![cfg(target_os = "linux")]` and is
silently skipped.

> **Want to write more?** S012 (TLS interception) names a few that should
> exist next: `intercept_e2e.rs`, `intercept_logging.rs`,
> `mixed_modes_e2e.rs`. Drop them in the same directory; `cargo test`
> picks them up.

## Continuous integration

`application/.github/workflows/ci.yml` runs four jobs on every push and PR
to `main`:

| Job | Command | What fails it |
|---|---|---|
| `check` | `cargo check --workspace --all-targets` | compilation error |
| `test` | `cargo test --workspace --all-targets` | a unit or integration test fails |
| `fmt` | `cargo fmt --all -- --check` | formatting drift |
| `clippy` | `cargo clippy --workspace --all-targets -- -D warnings` | any new clippy warning |

`-- -D warnings` on clippy is strict: a single new warning is treated as a
compilation error. Keep new code lint-clean.

> CI runs on `ubuntu-latest`, so the Linux-only integration test in
> `outcalld/tests/bridge_integration.rs` runs there but is gated by a
> root check. By default the test exits clean if it isn't running as
> root, so on a stock GitHub runner it's effectively a no-op.

## Code coverage

The Outcall workspace plays well with `cargo-llvm-cov`, which uses LLVM's
source-based coverage to produce per-file line coverage:

```sh
cargo install cargo-llvm-cov
cargo install cargo-nextest    # optional but faster

cd application

# Plain text summary
cargo llvm-cov --workspace --all-targets

# Per-file HTML report (open target/llvm-cov/html/index.html)
cargo llvm-cov --workspace --all-targets --html

# Just the daemon, including its integration test
cargo llvm-cov -p outcalld

# CI-friendly: emit lcov for upload
cargo llvm-cov --workspace --all-targets --lcov --output-path lcov.info
```

`cargo llvm-cov` recompiles with `-C instrument-coverage` then runs the
tests; expect a fresh first run to take 1–2 minutes longer than a normal
`cargo test`.

### Realistic coverage targets

Outcall is a network daemon — large parts of it are I/O, syscalls, and
async glue that is hard to unit-test. Aim for:

| Crate | Target line coverage | Why |
|---|---|---|
| `outcall-api` | 90%+ | Pure types and constants. Easy. |
| `outcalld/rules/` | 80%+ | Pure-ish CEL evaluation; should be heavily covered. |
| `outcalld/proxy/` (parsing) | 85%+ | The parser functions are pure; the IO loop isn't. |
| `outcalld/proxy/` (handle_*) | not unit-test territory | Use integration tests (S011 names a few). |
| `outcalld/network/`, `outcalld/dns/`, `outcalld/docker/` | covered via integration | Wire them into `tests/*.rs` rather than mocking everything. |

Don't chase a single workspace-wide percentage — the meaningful number is
"the parsers and the rule engine are well-covered, and every subsystem
has at least one integration test that proves the wiring works".

## End-to-end harness (`make test` / `make test-e2e`)

The `Makefile` at the repo root drives a Docker-based smoke test. This is
not unit testing — it's a "does the whole binary actually do the thing on a
fresh host" check.

```sh
make build          # one-time: build the outcall-e2e Docker image (~2 min)
make start          # creates network, starts outcalld in a container
make test           # runs HTTP / ICMP / DNS smoke tests against an Alpine agent
make test-e2e       # full E2E test suite from scripts/e2e/tests/
make stop           # tear everything down
```

| Make target | What it does |
|---|---|
| `make build` | Build `outcall-e2e` image |
| `make start` / `make stop` | Daemon lifecycle in Docker |
| `make status` | `outcall bridge status` inside the daemon container |
| `make agent` | Interactive Alpine shell on the outcall network |
| `make logs` | Tail daemon logs |
| `make exec CMD="…"` | Run any command inside the daemon container |
| `make clean` | Stop + remove the image |

`make test-e2e` is self-contained: it builds the image if needed and runs
the test scripts in `scripts/e2e/tests/` with the right capabilities
(`NET_ADMIN`, `NET_RAW`, `SYS_ADMIN`, `net.ipv4.ip_forward=1`).

### Adding an E2E test

Drop a numbered `.sh` script in `scripts/e2e/tests/`. Each script gets:

| Variable | Value | Description |
|---|---|---|
| `BRIDGE` | `outcall0` | Bridge interface name |
| `BRIDGE_IP` | `10.99.0.1` | Bridge IP |
| `AGENT_NS` | `agent1` | Network namespace name |
| `AGENT_IP` | `10.99.0.2` | Agent IP inside the namespace |
| `TARGET_IP` | (dynamic) | Container's eth0 IP (forwarded target) |

Exit `0` = pass, non-zero = fail. Existing scripts are documented in
[the test plans](/docs/guides/tests).

## Where to dig deeper

- [S012: Test Coverage](/docs/specs/012-test-coverage) — current coverage
  inventory, gaps, and the integration tests that should land.
- [Specs S001 / S003 / S006 / S011](/docs/specs) — every spec has its own
  acceptance scenarios and success criteria. Tests should cite the spec
  IDs they cover.
- [CLI reference](/docs/guides/cli) — the surface tested by these layers.
