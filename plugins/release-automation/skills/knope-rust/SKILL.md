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
versioned_files = ["crates/my-app/Cargo.toml", "Cargo.lock"]
changelog = "CHANGELOG.md"
assets = "artifacts/*"

[github]
owner = "username"
repo = "repo-name"

[[workflows]]
name = "prepare-release"

[[workflows.steps]]
type = "Command"
command = "git switch -c release"

[[workflows.steps]]
type = "PrepareRelease"

[[workflows.steps]]
type = "Command"
command = "git commit -m 'chore: prepare release' --signoff --all"

[[workflows.steps]]
type = "Command"
command = "git push --force --set-upstream origin release"

[[workflows.steps]]
type = "CreatePullRequest"
base = "main"
title = "chore: release $version"
body = "$changelog"

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

For multi-platform builds (Linux, macOS, Windows), load `references/multi-platform-release.md`.

### Step 4 — Verify

- Ensure `knope.toml` points to the correct `Cargo.toml` for versioning
- Confirm `fetch-depth: 0` is set on checkout steps that precede knope commands
- Verify `knope-dev/action` is SHA-pinned with version comment (see Quick Reference)
- Check artifact upload/download version pairing (upload@v6 + download@v7)
- Confirm skip condition is present on `prepare-release` to prevent infinite loops

## Quick Reference

### Correct Action Versions

| Action | Version | SHA Pin |
|:-------|:--------|:--------|
| `knope-dev/action` | `v2.1.0` | `407e9ef7c272d2dd53a4e71e39a7839e29933c48` |
| `actions/checkout` | `v6.0.2` | `de0fac2e4500dabe0009e67214ff5f5447ce83dd` |
| `actions/upload-artifact` | `v6.0.0` | `b7c566a772e6b6bfb58ed0dc250532a479d7789f` |
| `actions/download-artifact` | `v7.0.0` | `37930b1c2abaa49bbe596cd826c3c89aef350131` |
| `step-security/harden-runner` | `v2.14.2` | `5ef0c079ce82195b2a36a210272d6b661572d83e` |
| `dtolnay/rust-toolchain` | `@stable` | Floating tag is fine |
| `Swatinem/rust-cache` | `@v2` | Floating tag is fine |

### Knope Binary Version

Pin via the `version` parameter: `version: 0.22.2`

### Conventional Commit Mapping

| Prefix | Bump | Example |
|:-------|:-----|:--------|
| `feat:` | minor | `feat: add voice typing` |
| `fix:` | patch | `fix: handle empty audio` |
| `feat!:` or `fix!:` | major | `feat!: redesign API` |
| `chore:`, `ci:`, `docs:` | none | Not included in changelog |

## Anti-Patterns

- **`knope-dev/action@v2`** — Does not exist. Unlike official `actions/*` repos, `knope-dev/action` only publishes exact semver tags. Always use the full version like `@v2.1.0`, and prefer SHA pinning with a version comment.
- **Missing `fetch-depth: 0`** — Knope needs full git history to parse conventional commits since the last release tag. Shallow clones produce incomplete changelogs or version errors.
- **Wrong version source** — For Tauri apps, `versioned_files` must point to the Tauri crate's `Cargo.toml`, not the workspace root. Tauri reads its version from the crate, not from `tauri.conf.json`.
- **`download-artifact@v5` or `@v6`** — Use `download-artifact@v7` to pair with `upload-artifact@v6`. The v6/v7 pairing is the current correct combination.
- **Skipping bootstrap** — The first `prepare-release` run won't create a PR unless there are conventional commits after a release tag. Bootstrap by creating a `v0.1.0` tag on main first.
- **`CreatePullRequest` with `variables` map** — Knope 0.22.2 supports `$version` and `$changelog` built-in syntax directly in `title` and `body` strings. Use `title = "chore: release $version"` and `body = "$changelog"` instead of the verbose `[workflows.steps.title]` table with `variables` map.
- **Missing skip condition** — Without `if: "!contains(github.event.head_commit.message, 'chore: prepare release')"` on the prepare-release job, the workflow triggers on its own changelog commit, causing an infinite loop.

## Reference Guide

Load these files from the `references/` directory for detailed specifications:

| File | Contains | Load When |
|:-----|:---------|:----------|
| `workflow-templates.md` | Complete GitHub Actions workflow files for prepare-release and release | Creating or modifying CI workflows |
| `multi-platform-release.md` | Multi-platform build matrix, crates.io publishing, security hardening | Building for Linux/macOS/Windows or publishing to crates.io |
| `gotchas.md` | Version-specific pitfalls, security notes, bootstrapping instructions | Debugging CI failures or reviewing workflow security |
