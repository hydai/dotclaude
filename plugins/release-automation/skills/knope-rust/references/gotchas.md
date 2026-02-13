# Knope Release Automation — Gotchas

Hard-won pitfalls discovered through real-world setup. Check this list when debugging CI failures or reviewing workflow security.

## 1. knope-dev/action Requires Full Semver Tags

**Wrong:**
```yaml
- uses: knope-dev/action@v2  # Does NOT exist
```

**Correct:**
```yaml
- uses: knope-dev/action@v2.1.0
```

Unlike official GitHub Actions (e.g., `actions/checkout@v6`) which maintain floating major version tags (`v6` always points to the latest `v6.x.x`), the `knope-dev/action` repository only publishes exact semver tags. There is no `v2` tag — only `v2.1.0`, `v2.0.0`, etc.

This is actually a feature: you get pinned reproducibility instead of implicit upgrades. But it means you must check the [knope-dev/action releases page](https://github.com/knope-dev/action/releases) for the latest tag.

## 2. fetch-depth: 0 Is Required for Knope Commands

Knope parses the full git commit history to:
- Find commits since the last release tag
- Parse conventional commit prefixes (`feat:`, `fix:`, etc.)
- Determine the version bump and generate changelog entries

Without `fetch-depth: 0`, GitHub Actions performs a shallow clone (depth 1), and knope will either:
- Find no commits to include
- Produce an incomplete changelog
- Fail to determine the correct version bump

**Required in:** Any checkout step that precedes `knope prepare-release` or `knope release`.

**Not required in:** Build-only jobs that don't run knope commands.

## 3. Artifact Upload/Download Version Pairing

Use this combination:
- `actions/upload-artifact@v6` (upload)
- `actions/download-artifact@v5` (download)

The v6 upload API is cross-compatible with the v5 download API. Using `download-artifact@v6` can cause compatibility issues. This is the safe, tested pairing.

## 4. Version Source for Tauri Apps

For Tauri 2 apps, `versioned_files` in `knope.toml` must point to the **Tauri crate's** `Cargo.toml`:

```toml
[package]
versioned_files = ["crates/lt-tauri/Cargo.toml"]
```

Tauri 2 reads the app version from the crate's `Cargo.toml`, not from `tauri.conf.json` (which has no `version` field). If you point `versioned_files` at the workspace root `Cargo.toml`, the Tauri app won't pick up the version bump.

## 5. github.head_ref in if: Expressions Is Safe

Security scanners and hooks may flag `github.head_ref` as a risky input for GitHub Actions injection. However, the usage in release workflows:

```yaml
if: github.head_ref == 'release' && github.event.pull_request.merged == true
```

is **safe** because:
- It's evaluated in a GitHub Actions `if:` expression context
- It is NOT interpolated into a `run:` shell block
- GitHub Actions expressions are evaluated by the runner, not by bash
- There is no shell injection vector

The concern applies only when `github.head_ref` is used inside `run:` blocks like `run: echo "${{ github.head_ref }}"`, where a malicious branch name could inject shell commands.

## 6. Bootstrapping the First Release

After initial setup, the `prepare-release` workflow runs on the next push to main but won't create a release PR unless there are conventional commits (`feat:`, `fix:`) since the **last release tag**.

If the project has no release tags yet:
1. Create an initial tag: `git tag v0.1.0 && git push --tags`
2. Make a conventional commit: `git commit -m "feat: initial release"`
3. Push to main — knope will now create the first release PR

Without the bootstrap tag, knope may process the entire commit history or produce unexpected results.

## 7. Knope Binary Version Pinning

The `knope-dev/action` installs a specific knope binary version via the `version` parameter:

```yaml
- uses: knope-dev/action@v2.1.0
  with:
    version: 0.22.1
```

The **action tag** (`v2.1.0`) and the **knope binary version** (`0.22.1`) are independent. The action tag controls the installer script; the version parameter controls which knope release gets downloaded. Pin both for reproducible CI.

## 8. Permissions Block

The `prepare-release` workflow needs both:
```yaml
permissions:
  contents: write
  pull-requests: write
```

Without `pull-requests: write`, knope cannot create the release PR. Without `contents: write`, it cannot push the changelog updates.

The `release` workflow only needs `contents: write` (to create the GitHub Release).
