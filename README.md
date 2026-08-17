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

## Versioning

Consumers should pin to a major version tag (e.g. `@v1`), which is moved
forward to point at the latest compatible release. Breaking changes bump the
major tag (`v2`, ...).
