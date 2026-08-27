# Testing

Outcall has three layers of automated tests:

| Layer | Where | When you run it |
|---|---|---|
| **Unit and binary tests** | `src/**/*.rs`, `outcall/src/main_tests.rs`, and crate-local test modules | every `cargo test` |
| **Integration tests** | `outcall*/tests/*.rs` (portable plus Linux/privileged suites) | every `cargo test`; privileged cases are explicitly ignored |
| **End-to-end harness** | `Makefile` + `scripts/e2e/tests/*.sh` (Docker-based) | `make test` / `make test-e2e` |

This guide walks each layer in order. The first two are what most people
mean by "tests"; the third is a cross-binary smoke harness used to confirm
the whole stack lights up on a fresh Docker host.

## Unit tests (`cargo test`)

Run the whole workspace's tests from `application/`:

```sh
cd application
cargo test --workspace --all-targets --locked
```

The portable macOS all-target run currently executes **184 non-ignored
tests**. Linux executes additional daemon modules and target-gated tests. The
suite covers API serialization, CLI parsing and first-run behavior, rules,
policy editing, recipes, host-resource boundaries, the local UI bridge, and
daemon subsystems.

Most non-ignored tests need no privileges. Some portable tests create
loopback TCP or Unix sockets. Linux-only tests that require network
administration are marked `#[ignore]` and run in the privileged CI jobs.

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

Integration tests live in `outcall/tests/`, `outcall-api/tests/`, and
`outcalld/tests/`. They are separate test binaries and exercise public APIs,
CLI behavior, sockets, and real syscalls as appropriate.

`outcalld/tests/` currently contains **10** integration test files:

| File | What it exercises | Requirements |
|---|---|---|
| `bridge_integration.rs` | Bridge create/destroy, nftables apply/teardown | Linux + root |
| `cli_integration.rs` | CLI subcommands over Unix socket | outcalld running |
| `agent_api_integration.rs` | Agent shim verdict round-trip via agent.sock | Linux |
| `proxy_http_integration.rs` | HTTP proxy ALLOW/BLOCK | outcalld + bridge up |
| `proxy_https_integration.rs` | HTTPS CONNECT + SNI-based BLOCK | outcalld + bridge up |
| `proxy_dns_integration.rs` | DNS filter + proxy interaction | outcalld + bridge up |
| `dynamic_rules_integration.rs` | Dynamic rule insert + flush | outcalld + bridge up |
| `example_rules_validation.rs` | Validates the shipped example rulesets | None |
| `intercept_e2e.rs` | Rejects intercept rules without a CA; verifies non-intercept startup | Linux + root |
| `mixed_modes_e2e.rs` | Mixed-mode behavior for currently implemented egress modes | Linux + root |

The bridge test needs Linux and `CAP_NET_ADMIN` (or root):

```sh
sudo cargo test -p outcalld --test bridge_integration -- --nocapture
```

On macOS, Linux-gated files compile to zero tests. Privileged tests are also
ignored during the ordinary Linux run and execute in dedicated CI jobs.

> **Want to write more?** Drop a new `.rs` file in `outcalld/tests/` and
> `cargo test` picks it up automatically. See S012 for gaps in coverage.

## Continuous integration

`application/.github/workflows/ci.yml` runs the following job groups on pushes
and pull requests to `main`:

| Job group | Command or scope | What fails it |
|---|---|---|
| `check` | `cargo check --workspace --all-targets` | compilation error |
| `test-unit`, `test-integration` | portable unit/binary and integration suites | any non-ignored test failure |
| `test-privileged-sudo`, `test-privileged-docker` | ignored Linux integration suites with required capabilities | privileged bridge/proxy/runtime failure |
| `installer-smoke` | local release install plus Claude and Codex `--version` runs | packaging or first-run failure |
| `secure-install-runtime` | local install, both recipes, isolation/profile scripts, netfilter fail-closed test | runtime bootstrap or security regression |
| `coverage` | `make coverage` | tests fail or workspace line coverage drops below 50% |
| `spec-traceability` | `make spec-check` | an S000-S015 implementation/test mapping is missing or stale |
| `fmt` | `cargo fmt --all -- --check` | formatting drift |
| `clippy` | `cargo clippy --workspace --all-targets -- -D warnings` | any new clippy warning |
| `audit`, `deny`, `unsafe-policy` | dependency and first-party safety policy | advisory, license, ban, or unsafe-policy violation |

`-- -D warnings` on clippy is strict: a single new warning is treated as a
compilation error. Keep new code lint-clean.

The ordinary jobs do not imply privileged coverage. The sudo and Docker jobs
explicitly run ignored tests with the required Linux capabilities.

## Code coverage

The Outcall workspace plays well with `cargo-llvm-cov`, which uses LLVM's
source-based coverage to produce per-file line coverage:

```sh
cargo install cargo-llvm-cov

cd application

# Run tests, write LCOV, and enforce the current CI floor
make coverage

# Per-file HTML report (open target/llvm-cov/html/index.html)
cargo llvm-cov --workspace --all-targets --locked --html

# Just the daemon, including its integration test
cargo llvm-cov -p outcalld --all-targets --locked
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

CI uses a 50% workspace line floor as a regression guard and uploads
`target/coverage/lcov.info`. It does not replace the higher subsystem targets
in S012 or the requirement for real integration tests at trust boundaries.

## End-to-end harness (`make test` / `make test-e2e`)

The `Makefile` at the repo root drives a Docker-based smoke test. This is
not unit testing — it's a "does the whole binary actually do the thing on a
fresh host" check.

```sh
make build          # one-time: build the outcall-daemon Docker image (~2 min)
make start          # creates network, starts outcalld in a container
make test           # runs HTTP / ICMP / DNS smoke tests against an Alpine agent
make test-e2e       # full E2E test suite from scripts/e2e/tests/
make stop           # tear everything down
```

| Make target | What it does |
|---|---|
| `make build` | Build `outcall-daemon` image |
| `make start` / `make stop` | Daemon lifecycle in Docker |
| `make status` | `outcall bridge status` inside the daemon container |
| `make agent` | Interactive Alpine shell on the outcall network |
| `make logs` | Tail daemon logs |
| `make exec CMD="…"` | Run any command inside the daemon container |
| `make clean` | Stop + remove the image |

`make test-e2e` is self-contained: it builds the image if needed and runs
the test scripts in `scripts/e2e/tests/` with the right capabilities
(`NET_ADMIN`, `NET_RAW`, `SYS_ADMIN`, `net.ipv4.ip_forward=1`).

### Specialized suites

Beyond the numbered E2E suite, the `Makefile` ships focused suites that each
build the image if needed and run a dedicated script under `scripts/`:

| Make target | What it does |
|---|---|
| `make test-bypass` | Security bypass suite — attempts to escape the egress controls |
| `make test-payloads` | Payload attack suite — malformed protocol attacks |
| `make test-tls-intercept` | TLS interception tests (requires a CA + `mode: intercept` configured) |

### Adding an E2E test

Drop a numbered `.sh` script in `scripts/e2e/tests/`. Each script gets:

| Variable | Value | Description |
|---|---|---|
| `BRIDGE` | `outcall0` | Bridge interface name |
| `BRIDGE_IP` | `10.99.0.1` | Bridge IP |
| `AGENT_NS` | `agent1` | Network namespace name |
| `AGENT_IP` | `10.99.0.2` | Agent IP inside the namespace |
| `TARGET_IP` | (dynamic) | Container's eth0 IP (forwarded target) |

Exit `0` = pass, non-zero = fail. The existing E2E scripts live in
[`outcall-dev/root/scripts/e2e/tests/`](https://github.com/outcall-dev/root/tree/main/scripts/e2e/tests).

## Where to dig deeper

- [S012: Test Coverage](/docs/specs/012-test-coverage) — current coverage
  inventory, gaps, and the integration tests that should land.
- [Specs S001 / S003 / S006 / S011](/docs/specs) — every spec has its own
  acceptance scenarios and success criteria. Tests should cite the spec
  IDs they cover.
- [CLI reference](/docs/guides/cli) — the surface tested by these layers.
