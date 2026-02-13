# Knope Release Automation — Gotchas

Hard-won pitfalls discovered through real-world setup. Check this list when debugging CI failures or reviewing workflow security.

## 1. knope-dev/action Requires Full Semver Tags and SHA Pinning

**Wrong:**
```yaml
- uses: knope-dev/action@v2  # Does NOT exist
```

**Better (full semver):**
```yaml
- uses: knope-dev/action@v2.1.0
```

**Best (SHA-pinned with version comment):**
```yaml
- uses: knope-dev/action@407e9ef7c272d2dd53a4e71e39a7839e29933c48 # v2.1.0
  with:
    version: 0.22.2
```

Unlike official GitHub Actions (e.g., `actions/checkout@v6`) which maintain floating major version tags, the `knope-dev/action` repository only publishes exact semver tags. There is no `v2` tag — only `v2.1.0`, `v2.0.0`, etc.

SHA pinning is the recommended approach for production workflows. It provides reproducibility and protection against tag mutation attacks. Add a version comment for human readability.

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
- `actions/upload-artifact@v6` (upload) — SHA: `b7c566a772e6b6bfb58ed0dc250532a479d7789f`
- `actions/download-artifact@v7` (download) — SHA: `37930b1c2abaa49bbe596cd826c3c89aef350131`

The v6 upload API pairs with the v7 download API. Using `download-artifact@v5` or `@v6` with `upload-artifact@v6` can cause compatibility issues. The v6 upload + v7 download is the tested, safe pairing.

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
- uses: knope-dev/action@407e9ef7c272d2dd53a4e71e39a7839e29933c48 # v2.1.0
  with:
    version: 0.22.2
```

The **action tag** (`v2.1.0`) and the **knope binary version** (`0.22.2`) are independent. The action tag controls the installer script; the version parameter controls which knope release gets downloaded. Pin both for reproducible CI.

## 8. Permissions Block

The `prepare-release` workflow needs both:
```yaml
permissions:
  contents: write
  pull-requests: write
```

Without `pull-requests: write`, knope cannot create the release PR. Without `contents: write`, it cannot push the changelog updates.

The `release` workflow only needs `contents: write` (to create the GitHub Release).

## 9. CreatePullRequest Uses $version and $changelog Built-ins

Knope 0.22.2 supports `$version` and `$changelog` built-in syntax directly in `title` and `body` strings.

**Outdated — verbose variables map:**
```toml
[[workflows.steps]]
type = "CreatePullRequest"
base = "main"

[workflows.steps.title]
template = "chore: release {version}"
variables = { "{version}" = "Version" }

[workflows.steps.body]
template = "{changelog}"
variables = { "{changelog}" = "ChangelogEntry" }
```

**Current — built-in syntax:**
```toml
[[workflows.steps]]
type = "CreatePullRequest"
base = "main"
title = "chore: release $version"
body = "$changelog"
```

The `$version` and `$changelog` built-ins replace the need for the `template`/`variables` table syntax. The result is identical but the config is cleaner.

## 10. Infinite Loop Prevention

The `prepare-release` workflow pushes a changelog commit to main, which triggers the workflow again. Without a skip condition, this creates an infinite loop.

**Required skip condition on the prepare-release job:**
```yaml
if: "!contains(github.event.head_commit.message, 'chore: prepare release')"
```

This checks if the commit that triggered the workflow is itself a prepare-release commit, and skips if so. The message must match the commit message used in the `Command` step of `knope.toml`:

```toml
[[workflows.steps]]
type = "Command"
command = "git commit -m 'chore: prepare release' --signoff --all"
```

## 11. Git User Configuration

GitHub Actions runners have no git user identity by default. Knope's `Command` steps that run `git commit` will fail without configuring one.

**Required before any git commit steps:**
```yaml
- name: Configure git user
  run: |
    git config user.name 'github-actions[bot]'
    git config user.email '41898282+github-actions[bot]@users.noreply.github.com'
```

The `41898282+` prefix is the numeric user ID for the `github-actions[bot]` account. Using this ID ensures commits are properly attributed to the bot in GitHub's UI.

## 12. continue-on-error for Graceful No-Op

When there are no conventional commits since the last release, `knope prepare-release` exits with a non-zero status. This fails the workflow even though "nothing to release" is a valid outcome.

**Add `continue-on-error` and `--verbose` to the prepare-release step:**
```yaml
- name: Run knope prepare-release
  run: knope prepare-release --verbose
  continue-on-error: true
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The `--verbose` flag provides detailed output for debugging when things do go wrong. Combined with `continue-on-error`, the workflow succeeds cleanly when there's nothing to release while still logging what knope evaluated.
