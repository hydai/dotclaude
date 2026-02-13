# Multi-Platform Release Workflow

Complete workflow template for building Rust CLI binaries across Linux, macOS, and Windows, publishing to GitHub Releases and crates.io. Based on production patterns from wasmedgeup.

## 1. Multi-Platform Build Matrix

Build binaries for 7 targets using a matrix strategy with cross-compilation.

```yaml
name: Release

on:
  pull_request:
    types: [closed]
  workflow_dispatch:

permissions: {}

jobs:
  build:
    if: |
      (github.head_ref == 'release' && github.event.pull_request.merged == true)
      || github.event_name == 'workflow_dispatch'
    strategy:
      fail-fast: true
      matrix:
        include:
          - target: x86_64-unknown-linux-gnu
            os: ubuntu-latest
          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
          - target: aarch64-unknown-linux-gnu
            os: ubuntu-latest
          - target: aarch64-unknown-linux-musl
            os: ubuntu-latest
          - target: x86_64-apple-darwin
            os: macos-latest
          - target: aarch64-apple-darwin
            os: macos-latest
          - target: x86_64-pc-windows-msvc
            os: windows-latest
    runs-on: ${{ matrix.os }}
    steps:
      - name: Harden runner
        uses: step-security/harden-runner@5ef0c079ce82195b2a36a210272d6b661572d83e # v2.14.2
        with:
          egress-policy: audit

      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2

      - name: Setup cross-compilation toolchain
        uses: taiki-e/setup-cross-toolchain-action@b8d1a322a6009a2b7220f53996695778eef89b41 # v1
        with:
          target: ${{ matrix.target }}

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Rust cache
        uses: Swatinem/rust-cache@v2

      - name: Build release binary
        run: cargo build --release --target ${{ matrix.target }}

      - name: Package artifact (Unix)
        if: runner.os != 'Windows'
        run: |
          BINARY_NAME=my-app
          tar -czf ${BINARY_NAME}-${{ matrix.target }}.tgz -C target/${{ matrix.target }}/release ${BINARY_NAME}

      - name: Package artifact (Windows)
        if: runner.os == 'Windows'
        shell: pwsh
        run: |
          $BINARY_NAME = "my-app"
          Compress-Archive -Path "target/${{ matrix.target }}/release/${BINARY_NAME}.exe" -DestinationPath "${BINARY_NAME}-${{ matrix.target }}.zip"

      - name: Upload artifact
        uses: actions/upload-artifact@b7c566a772e6b6bfb58ed0dc250532a479d7789f # v6.0.0
        with:
          name: ${{ matrix.target }}
          path: |
            *.tgz
            *.zip
          if-no-files-found: error
```

**Customization:**
- Replace `my-app` with your binary name (the `[[bin]]` name from `Cargo.toml`)
- Adjust the matrix targets for your supported platforms
- Add cross-compilation dependencies if needed for specific targets

## 2. Release Job

Downloads all build artifacts and runs `knope release` to create the GitHub Release with attached binaries.

```yaml
  release:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Harden runner
        uses: step-security/harden-runner@5ef0c079ce82195b2a36a210272d6b661572d83e # v2.14.2
        with:
          egress-policy: audit

      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          fetch-depth: 0

      - uses: actions/download-artifact@37930b1c2abaa49bbe596cd826c3c89aef350131 # v7.0.0
        with:
          path: artifacts/
          merge-multiple: true

      - uses: knope-dev/action@407e9ef7c272d2dd53a4e71e39a7839e29933c48 # v2.1.0
        with:
          version: 0.22.2

      - name: Run knope release
        run: knope release --verbose
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Key points:**
- `merge-multiple: true` on download-artifact flattens all matrix artifacts into a single `artifacts/` directory
- `fetch-depth: 0` is required for knope to parse the full git history
- The `assets = "artifacts/*"` in `knope.toml` tells knope to attach all files in that directory to the GitHub Release

## 3. Crates.io Publishing

Optional job to publish the crate to crates.io using OIDC token authentication (no long-lived API token needed).

```yaml
  publish-crate:
    needs: release
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Harden runner
        uses: step-security/harden-runner@5ef0c079ce82195b2a36a210272d6b661572d83e # v2.14.2
        with:
          egress-policy: audit

      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable

      - name: Authenticate with crates.io
        uses: rust-lang/crates-io-auth-action@041cce5b4b821e6b0ebc9c9c38b58cac4e34dcc2 # v1.0.2

      - name: Publish to crates.io
        run: cargo publish
```

**Requirements:**
- `id-token: write` permission is required for OIDC authentication
- The crate must be configured for trusted publishing on crates.io (see [crates.io trusted publishing docs](https://doc.rust-lang.org/cargo/reference/registry-authentication.html))
- `rust-lang/crates-io-auth-action` handles OIDC token exchange — no `CARGO_REGISTRY_TOKEN` secret needed

## 4. Security Hardening

Every job in the workflow should include `step-security/harden-runner`:

```yaml
- name: Harden runner
  uses: step-security/harden-runner@5ef0c079ce82195b2a36a210272d6b661572d83e # v2.14.2
  with:
    egress-policy: audit
```

This must be the **first step** in every job. It monitors network egress during the job and can be configured to block unauthorized outbound connections, providing defense against supply-chain attacks where compromised dependencies attempt to exfiltrate data.

**Egress policies:**
- `audit` — Logs all outbound connections (recommended starting point)
- `block` — Blocks all outbound connections not in an allow list (strict mode)
