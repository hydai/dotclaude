# GitHub Actions Workflow Templates

Complete, copy-paste-ready workflow templates for knope-based release automation. Customize the build job for your specific project.

## prepare-release.yml

Runs on every push to main. Parses conventional commits and creates a release PR with changelog updates.

```yaml
name: Prepare Release

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  prepare-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: knope-dev/action@v2.1.0
        with:
          version: 0.22.1

      - run: knope prepare-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Key points:**
- `fetch-depth: 0` is required — knope reads full git history to find commits since the last release tag
- `permissions` must include both `contents: write` and `pull-requests: write`
- `GITHUB_TOKEN` is automatically provided by GitHub Actions

## release.yml — Rust-Only Variant

For projects that only need to publish a GitHub Release (no platform-specific builds).

```yaml
name: Release

on:
  pull_request:
    types: [closed]

jobs:
  release:
    if: github.head_ref == 'release' && github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: knope-dev/action@v2.1.0
        with:
          version: 0.22.1

      - run: knope release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## release.yml — Tauri macOS DMG Variant

For Tauri apps that build a macOS universal binary (.dmg) and attach it to the GitHub Release.

```yaml
name: Release

on:
  pull_request:
    types: [closed]

jobs:
  build:
    if: github.head_ref == 'release' && github.event.pull_request.merged == true
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v6

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: aarch64-apple-darwin,x86_64-apple-darwin

      - name: Rust cache
        uses: Swatinem/rust-cache@v2

      - name: Install frontend dependencies
        working-directory: ui
        run: npm ci

      - name: Install Tauri CLI
        run: cargo install tauri-cli

      - name: Build universal binary
        run: cargo tauri build --target universal-apple-darwin

      - name: Upload DMG artifact
        uses: actions/upload-artifact@v6
        with:
          name: app-universal-dmg
          path: target/universal-apple-darwin/release/bundle/dmg/*.dmg
          if-no-files-found: error

  publish:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: knope-dev/action@v2.1.0
        with:
          version: 0.22.1

      - name: Run knope release
        run: knope release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - uses: actions/download-artifact@v5
        with:
          name: app-universal-dmg
          path: artifacts/

      - name: Upload DMG to GitHub Release
        run: |
          VERSION=$(grep '^version' crates/lt-tauri/Cargo.toml | head -1 | sed 's/.*"\(.*\)"/\1/')
          gh release upload "v${VERSION}" artifacts/*.dmg
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Customization points:**
- Replace `crates/lt-tauri/Cargo.toml` with the path to your versioned Cargo.toml
- Replace `app-universal-dmg` with a descriptive artifact name for your project
- Adjust `working-directory: ui` to match your frontend directory
- For non-Tauri Rust projects, remove the frontend and Tauri CLI steps

## The `if:` Gate Explained

Both release workflow variants use this condition:

```yaml
if: github.head_ref == 'release' && github.event.pull_request.merged == true
```

This ensures the release workflow only runs when:
1. The closed PR's source branch is named `release` (the branch knope creates)
2. The PR was actually merged (not closed without merging)

The `github.head_ref` usage here is safe — it's evaluated in a GitHub Actions expression context, not in a shell `run:` block, so there is no command injection vector.
