# CI/CD Pipeline

Outcall uses GitHub Actions for automated builds, releases, and container images.

## Workflows

### build-release.yml

Triggered when a version tag (`v*`) is pushed. Builds binaries and a multi-arch container image:

| Platform | Arch | Output |
|---|---|---|
| Linux | x64, arm64 | `outcall`, `outcalld` binaries |
| macOS | x64, arm64 | `outcall` (CLI only — `outcalld` is Linux-only) |

Note: `outcalld` requires Linux for bridge and nftables management (`#[cfg(target_os = "linux")]` gates the entire daemon main function). The macOS build produces the CLI client only.

Each artifact is published to the GitHub Release and accompanied by a `SHA256SUMS.txt`.

The container image is pushed to `ghcr.io/Outcall-dev/outcall` with the following tags:
- `vX.Y.Z` — exact version
- `sha-{sha}` — commit SHA
- `latest` — most recent tag

### tag-on-merge.yml

Triggered when a PR is merged to `main`. Automatically increments the patch version (e.g. `v0.1.0` → `v0.1.1`) and pushes the new tag, which in turn triggers `build-release.yml`.

### version-bump.yml

Manual workflow dispatch for minor and major version bumps. Opens a PR that updates `version` in `application/Cargo.toml`; on merge to `main`, `tag-on-merge.yml` handles the actual tag.

**Trigger**: GitHub → Actions → Version Bump → Run workflow

## Versioning

| Bump type | How to trigger |
|---|---|
| Patch | Merge any PR to `main` |
| Minor | `version-bump.yml` workflow → select `minor` |
| Major | `version-bump.yml` workflow → select `major` |

Each crate carries its own version in its `Cargo.toml` (e.g. `application/outcalld/Cargo.toml`). The workspace `application/Cargo.toml` does not declare a version — the `version-bump.yml` workflow updates individual crate manifests.

## Container Image

Built from `application/Dockerfile.test` with multi-arch support (`linux/amd64`, `linux/arm64`).

```bash
docker run ghcr.io/Outcall-dev/outcall:latest --help
```

## Secrets

| Secret | Required for |
|---|---|
| `GITHUB_TOKEN` | Always available (provided by GitHub Actions) |
| `GHCR_TOKEN` | Container image push — uses `GITHUB_TOKEN` by default |

No additional secrets are required for the standard release flow.