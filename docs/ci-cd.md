# CI/CD Pipeline

Outcall uses GitHub Actions for automated checks, release builds, and container
image publishing.

## Workflows

### build-release.yml

Triggered when a version tag (`v*`) is pushed. Builds release binaries and a
multi-arch container image:

| Platform | Arch | Output |
|---|---|---|
| Linux | x64, arm64 | `outcall`, `outcalld` binaries |
| macOS | x64, arm64 | `outcall`, `outcalld`, `outcall-agent`; the CLI runs the daemon in Docker Desktop's Linux VM |

Note: `outcalld` requires Linux for bridge and nftables management. macOS can
build the binary, but starting the daemon fails outside Linux.

Each archive is published to the GitHub Release with its own `.sha256` file.

The release also publishes Docker-loadable daemon image archives:
- `outcalld-image-linux-amd64.tar.gz`
- `outcalld-image-linux-arm64.tar.gz`

The public installer uses these archives to preload the matching Linux daemon
image during first-time Linux and macOS installs when Docker is available.

The container image is pushed to `ghcr.io/outcall-dev/outcalld` with these tags:
- `vX.Y.Z` — exact version
- `latest` — most recent tag

### tag-on-merge.yml

Triggered when a PR is merged to `main`. Automatically increments the patch version (e.g. `v0.1.0` → `v0.1.1`) and pushes the new tag, which in turn triggers `build-release.yml`.

### version-bump.yml

Manual workflow dispatch for minor and major version bumps. Opens a PR that updates `version` in `application/Cargo.toml`; on merge to `main`, `tag-on-merge.yml` handles the actual tag.

**Trigger**: GitHub → Actions → Version Bump → Run workflow

### ci.yml

Runs on pull requests and pushes to `main`. In addition to check, test, format,
Clippy, dependency policy, and unsafe-policy jobs, it includes:

- `coverage`: runs `make coverage`, enforces the workspace line floor, and
  uploads `coverage-lcov`;
- `spec-traceability`: validates the S000-S015 implementation/test map;
- `installer-smoke`: packages local artifacts and runs one-command Claude and
  Codex version smokes;
- `secure-install-runtime`: enforces `br_netfilter`, installs through
  `scripts/install.sh`, runs both recipes, validates container isolation and
  managed profiles, and proves netfilter preflight fails closed;
- privileged sudo and Docker jobs for ignored Linux integration tests.

The install/runtime jobs use Ubuntu 22.04 so host-built binaries remain
compatible with the Debian Bookworm daemon image.

## Versioning

| Bump type | How to trigger |
|---|---|
| Patch | Merge any PR to `main` |
| Minor | `version-bump.yml` workflow → select `minor` |
| Major | `version-bump.yml` workflow → select `major` |

Each crate carries its own version in its `Cargo.toml` (e.g. `application/outcalld/Cargo.toml`). The workspace `application/Cargo.toml` does not declare a version — the `version-bump.yml` workflow updates individual crate manifests.

## Container Image

Release images are built from `application/Dockerfile` with multi-arch support
for `linux/amd64` and `linux/arm64`.

```bash
docker run ghcr.io/outcall-dev/outcalld:latest --version
```

## Secrets

| Secret | Required for |
|---|---|
| `GITHUB_TOKEN` | Always available (provided by GitHub Actions) |

No additional secrets are required for the standard release flow.
