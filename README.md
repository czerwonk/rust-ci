# rust-ci

Shared, reusable GitHub Actions workflows for czerwonk's Rust projects.

## Workflows

### `ci.yml`

Runs `cargo clippy` (warnings as errors) and `cargo test` for the whole workspace.

```yaml
name: CI

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
  ci:
    uses: czerwonk/rust-ci/.github/workflows/ci.yml@v1
```

If a project needs to inject pre-built assets (e.g. bundled JS/WASM) before
running clippy/test, pass `artifact-name`/`artifact-path` (downloaded via
`actions/download-artifact` before the Rust steps) and `env` (a JSON object
of extra environment variables, e.g. to skip an asset build step):

```yaml
jobs:
  ci:
    needs: build-assets
    uses: czerwonk/rust-ci/.github/workflows/ci.yml@v1
    with:
      artifact-name: ${{ needs.build-assets.outputs.artifact-name }}
      artifact-path: server/includes/
      env: '{"SKIP_ASSET_BUILD": "1"}'
```

### `release.yml`

Cross-compiles a binary for a matrix of targets, packages it (`tar.gz` on
macOS/Linux, `zip` on Windows), and publishes a GitHub Release with the
artifacts attached.

```yaml
name: Release

on:
  push:
    tags: ["v*"]

jobs:
  release:
    uses: czerwonk/rust-ci/.github/workflows/release.yml@v1
    with:
      bin-name: my-binary
      matrix: |
        [
          {"release_for": "Linux-x86_64", "os": "ubuntu-latest", "target": "x86_64-unknown-linux-gnu"},
          {"release_for": "Linux-aarch64", "os": "ubuntu-latest", "target": "aarch64-unknown-linux-gnu"},
          {"release_for": "macOS-x86_64", "os": "macOS-latest", "target": "x86_64-apple-darwin"},
          {"release_for": "macOS-aarch64", "os": "macOS-latest", "target": "aarch64-apple-darwin"},
          {"release_for": "Windows-x86_64", "os": "windows-latest", "target": "x86_64-pc-windows-msvc"}
        ]
    secrets: inherit
```

`release.yml` accepts the same `artifact-name`/`artifact-path`/`env` inputs
as `ci.yml`, downloaded before the per-target `cargo build`.

### `docker.yml`

Builds and pushes a Docker image to `ghcr.io` using Buildx, with GHA layer
caching and cosign signing (skipped on pull requests).

```yaml
name: Docker

on:
  push:
    branches: ["main"]
    tags: ["v*"]
  pull_request:
    branches: ["main"]

jobs:
  docker:
    uses: czerwonk/rust-ci/.github/workflows/docker.yml@v1
    secrets: inherit
```

It also accepts the shared `artifact-name`/`artifact-path` inputs to inject
pre-built assets into the build context, plus `context` (default `.`),
`dockerfile` (default `Dockerfile`), `registry` (default `ghcr.io`),
`image-name` (default: the calling repository), `target` (stage of a
multi-stage Dockerfile), `build-args` (one `KEY=VALUE` per line), and
`sign-image` (default `true`) to opt out of cosign signing.

A repository that ships more than one image calls the workflow once per
image; `image-name` keeps the refs and the build caches apart:

```yaml
jobs:
  docker:
    uses: czerwonk/rust-ci/.github/workflows/docker.yml@v1
    secrets: inherit
  docker-worker:
    uses: czerwonk/rust-ci/.github/workflows/docker.yml@v1
    secrets: inherit
    with:
      image-name: czerwonk/my-project-worker
      target: worker
      build-args: |
        BIN=my-worker
```

### `publish.yml`

Publishes one or more crates to crates.io in the given order (useful for a
`lib` crate that a `cli`/`server` crate depends on), with an optional check
that a manifest's `version` matches the pushed `vX.Y.Z` tag.

```yaml
name: Publish

on:
  push:
    tags: ["v*"]

jobs:
  publish:
    uses: czerwonk/rust-ci/.github/workflows/publish.yml@v1
    secrets: inherit
    with:
      packages: '["my-lib", "my-cli"]'
      version-manifest: lib/Cargo.toml
```

Requires a `CARGO_REGISTRY_TOKEN` secret available to the caller (passed
through via `secrets: inherit`).

### `helm-release.yml`

Packages a Helm chart and publishes it to a `gh-pages`-style branch as a
Helm repository (index regenerated via `helm repo index`, chart committed
and pushed only if it changed).

```yaml
name: Release Helm Chart

on:
  push:
    branches: ["main"]
    paths: ["helm/**"]
  workflow_dispatch:

jobs:
  release:
    uses: czerwonk/rust-ci/.github/workflows/helm-release.yml@v1
    with:
      chart-path: helm/my-chart
      pages-url: https://my-org.github.io/my-repo
```

If the chart declares dependencies on external repos, pass
`dependency-repos` (a JSON object of repo name -> URL, added via `helm repo
add` before `helm dependency update`):

```yaml
    with:
      chart-path: helm/my-chart
      pages-url: https://my-org.github.io/my-repo
      dependency-repos: '{"bitnami": "https://charts.bitnami.com/bitnami"}'
```

`gh-pages-branch` (default `gh-pages`) overrides the branch the packaged
repository is hosted on.

## Versioning

Consumers should pin to a major version tag (e.g. `@v1`), which is moved
forward to point at the latest compatible release. Breaking changes bump the
major tag (`v2`, ...).
