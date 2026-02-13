---
name: knope-rust
description: >
  Use when setting up knope for Rust projects, implementing release automation
  with conventional commits, changelog generation, GitHub release publishing,
  or configuring knope.toml and GitHub Actions workflows for Rust/Tauri apps.
  Activates on mentions of knope, release automation, automated versioning,
  prepare-release, or conventional commit-based releases in Rust contexts.
---

# Knope Release Automation for Rust

## Role

You are a release automation engineer specializing in Rust projects. Your goal is to configure knope-based release workflows that follow conventional commits, generate changelogs, and publish GitHub Releases — correctly on the first try.

## Workflow

### Step 1 — Assess Project Structure

Before writing config, understand the project:
- **Workspace layout**: Single crate or Cargo workspace? Library or end-user app?
- **Version source**: Which `Cargo.toml` holds the canonical version? (For Tauri apps, this is the Tauri crate's `Cargo.toml`, not the root.)
- **Existing CI**: Any GitHub Actions already in place? What versions of `actions/checkout`, `upload-artifact`, etc.?
- **Build targets**: Platform-specific builds (macOS DMG, Linux AppImage, Windows MSI)?

### Step 2 — Configure knope.toml

Create `knope.toml` at the project root. Key decisions:

**Single package (app)** — all crates release together:
```toml
[package]
versioned_files = ["crates/my-app/Cargo.toml"]
changelog = "CHANGELOG.md"

[github]
owner = "username"
repo = "repo-name"

[[workflows]]
name = "prepare-release"

[[workflows.steps]]
type = "PrepareRelease"

[[workflows.steps]]
type = "CreatePullRequest"
base = "main"

[workflows.steps.title]
template = "chore: release {version}"
variables = { "{version}" = "Version" }

[workflows.steps.body]
template = "{changelog}"
variables = { "{changelog}" = "ChangelogEntry" }

[[workflows]]
name = "release"

[[workflows.steps]]
type = "Release"
```

**Workspace (library)** — each crate versioned independently:
```toml
[packages.my-core]
versioned_files = ["crates/my-core/Cargo.toml"]
changelog = "crates/my-core/CHANGELOG.md"

[packages.my-cli]
versioned_files = ["crates/my-cli/Cargo.toml"]
changelog = "crates/my-cli/CHANGELOG.md"
```

### Step 3 — Create GitHub Actions Workflows

Two workflows are needed:
1. **`prepare-release.yml`** — runs on push to main, creates a release PR with changelog updates
2. **`release.yml`** — runs when the release PR is merged, publishes the GitHub Release

Load the full workflow templates from `references/workflow-templates.md`.

### Step 4 — Verify

- Ensure `knope.toml` points to the correct `Cargo.toml` for versioning
- Confirm `fetch-depth: 0` is set on checkout steps that precede knope commands
- Verify `knope-dev/action` uses a full semver tag (see Anti-Patterns)
- Check artifact upload/download version pairing

## Quick Reference

### Correct Action Versions

| Action | Tag | Notes |
|:-------|:----|:------|
| `knope-dev/action` | `@v2.1.0` | **Must use full semver** — no floating `@v2` |
| `actions/checkout` | `@v6` | Floating major tag is fine |
| `actions/upload-artifact` | `@v6` | Floating major tag is fine |
| `actions/download-artifact` | `@v5` | Pairs with upload-artifact@v6 |
| `dtolnay/rust-toolchain` | `@stable` | For Rust builds |
| `Swatinem/rust-cache` | `@v2` | Floating major tag is fine |

### Knope Binary Version

Pin via the `version` parameter: `version: 0.22.1`

### Conventional Commit Mapping

| Prefix | Bump | Example |
|:-------|:-----|:--------|
| `feat:` | minor | `feat: add voice typing` |
| `fix:` | patch | `fix: handle empty audio` |
| `feat!:` or `fix!:` | major | `feat!: redesign API` |
| `chore:`, `ci:`, `docs:` | none | Not included in changelog |

## Anti-Patterns

- **`knope-dev/action@v2`** — Does not exist. Unlike official `actions/*` repos, `knope-dev/action` only publishes exact semver tags. Always use the full version like `@v2.1.0`.
- **Missing `fetch-depth: 0`** — Knope needs full git history to parse conventional commits since the last release tag. Shallow clones produce incomplete changelogs or version errors.
- **Wrong version source** — For Tauri apps, `versioned_files` must point to the Tauri crate's `Cargo.toml`, not the workspace root. Tauri reads its version from the crate, not from `tauri.conf.json`.
- **`download-artifact@v6`** — Use `@v5` instead. The v6 upload is cross-compatible with v5 download; using v6 for both can cause issues.
- **Skipping bootstrap** — The first `prepare-release` run won't create a PR unless there are conventional commits after a release tag. Bootstrap by creating a `v0.1.0` tag on main first.
- **Bare `CreatePullRequest` without `title`/`body`** — Knope 0.22.1 requires structured `title` and `body` objects with `template` and `variables`. Omitting them causes `knope prepare-release` to fail.

## Reference Guide

Load these files from the `references/` directory for detailed specifications:

| File | Contains | Load When |
|:-----|:---------|:----------|
| `workflow-templates.md` | Complete GitHub Actions workflow files for prepare-release and release | Creating or modifying CI workflows |
| `gotchas.md` | Version-specific pitfalls, security notes, bootstrapping instructions | Debugging CI failures or reviewing workflow security |
