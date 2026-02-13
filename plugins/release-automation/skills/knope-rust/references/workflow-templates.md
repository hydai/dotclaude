# GitHub Actions Workflow Templates

Complete, copy-paste-ready workflow templates for knope-based release automation. All actions are SHA-pinned for reproducibility and security.

## prepare-release.yml

Runs on every push to main and via manual dispatch. Parses conventional commits and creates a release PR with changelog updates.

```yaml
name: Prepare Release

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions: {}

jobs:
  prepare-release:
    if: "!contains(github.event.head_commit.message, 'chore: prepare release')"
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - name: Harden runner
        uses: step-security/harden-runner@5ef0c079ce82195b2a36a210272d6b661572d83e # v2.14.2
        with:
          egress-policy: audit

      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          fetch-depth: 0

      - name: Configure git user
        run: |
          git config user.name 'github-actions[bot]'
          git config user.email '41898282+github-actions[bot]@users.noreply.github.com'

      - uses: knope-dev/action@407e9ef7c272d2dd53a4e71e39a7839e29933c48 # v2.1.0
        with:
          version: 0.22.2

      - name: Run knope prepare-release
        run: knope prepare-release --verbose
        continue-on-error: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Key points:**
- `permissions: {}` at workflow level disables all permissions by default; job-level block grants only what's needed
- Skip condition prevents infinite loops from the changelog commit
- `step-security/harden-runner` audits network egress for supply-chain security
- `fetch-depth: 0` is required — knope reads full git history to find commits since the last release tag
- Git user config is required for knope's `Command` steps that run `git commit`
- `continue-on-error: true` allows graceful no-op when no conventional commits exist
- `--verbose` flag provides detailed output for debugging
- `GITHUB_TOKEN` is automatically provided by GitHub Actions

## release.yml — Rust-Only Variant

For projects that only need to publish a GitHub Release (no platform-specific builds).

```yaml
name: Release

on:
  pull_request:
    types: [closed]
  workflow_dispatch:

permissions: {}

jobs:
  release:
    if: |
      (github.head_ref == 'release' && github.event.pull_request.merged == true)
      || github.event_name == 'workflow_dispatch'
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

      - uses: knope-dev/action@407e9ef7c272d2dd53a4e71e39a7839e29933c48 # v2.1.0
        with:
          version: 0.22.2

      - name: Run knope release
        run: knope release --verbose
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
  workflow_dispatch:

permissions: {}

jobs:
  build:
    if: |
      (github.head_ref == 'release' && github.event.pull_request.merged == true)
      || github.event_name == 'workflow_dispatch'
    runs-on: macos-latest
    steps:
      - name: Harden runner
        uses: step-security/harden-runner@5ef0c079ce82195b2a36a210272d6b661572d83e # v2.14.2
        with:
          egress-policy: audit

      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2

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
        uses: actions/upload-artifact@b7c566a772e6b6bfb58ed0dc250532a479d7789f # v6.0.0
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
      - name: Harden runner
        uses: step-security/harden-runner@5ef0c079ce82195b2a36a210272d6b661572d83e # v2.14.2
        with:
          egress-policy: audit

      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          fetch-depth: 0

      - uses: knope-dev/action@407e9ef7c272d2dd53a4e71e39a7839e29933c48 # v2.1.0
        with:
          version: 0.22.2

      - name: Run knope release
        run: knope release --verbose
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - uses: actions/download-artifact@37930b1c2abaa49bbe596cd826c3c89aef350131 # v7.0.0
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
if: |
  (github.head_ref == 'release' && github.event.pull_request.merged == true)
  || github.event_name == 'workflow_dispatch'
```

This ensures the release workflow runs when:
1. The closed PR's source branch is named `release` (the branch knope creates) **and** the PR was merged
2. **Or** the workflow was triggered manually via `workflow_dispatch`

The `github.head_ref` usage here is safe — it's evaluated in a GitHub Actions expression context, not in a shell `run:` block, so there is no command injection vector.
