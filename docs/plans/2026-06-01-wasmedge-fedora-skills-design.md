# WasmEdge Fedora/RHEL Packaging Skills — Design

> Status: **DRAFT — awaiting review.** Do not implement until approved.
> Date: 2026-06-01

## Overview

Add the first of several planned **WasmEdge maintainer skills** to this marketplace.
This first effort turns the "Upgrade WasmEdge Fedora Package" runbook into reusable,
model-invoked skills under a new generic `wasmedge` plugin.

Per the three design decisions made during brainstorming:

| Decision | Choice |
|:---------|:-------|
| Packaging | **Split into two skills** — Fedora (Rawhide + stable branches) and EPEL (8/9/10) |
| Audience | **Generalized with placeholders** — any WasmEdge maintainer can use it |
| Execution | **Guide, human runs interactive** — assistant supplies exact commands; the human runs credentialed/interactive steps |

## Why split Fedora from EPEL

The runbook is one long procedure, but it has a natural seam:

- **Fedora** (Rawhide + `f4x` stable branches) shares one base image (`fedora:rawhide` / `mock`),
  one credential setup, and the canonical per-version "spec-bump → build → ship" loop.
- **EPEL** (8/9/10) is a different world: CentOS Stream base images, different repo enabling
  (`powertools` vs `crb`), a branch-request step, and the big gotcha that `%autorelease`/`%autochangelog`
  are unavailable so the changelog must be written by hand.

Keeping them as two skills means each `SKILL.md` stays focused and under the repo's 2000-word
guideline, and the EPEL skill can reference the Fedora skill's canonical loop instead of duplicating it.

**Alternatives considered:** (a) one monolithic skill — rejected, would blow the word budget and bury
EPEL specifics; (b) a third shared "core-fedpkg" skill — rejected as over-engineering; instead the EPEL
skill points back to the Fedora skill's `rawhide-release.md` for the shared loop.

## Proposed file structure

```
plugins/wasmedge/
  .claude-plugin/
    plugin.json
  skills/
    fedora-package-upgrade/
      SKILL.md
      references/
        environment-setup.md     # container + maintainer credentials (one-time)
        rawhide-release.md        # CANONICAL per-version loop + bodhi/koji follow-up
        branched-release.md       # f4x stable branches (merge rawhide, mock, manual update)
    epel-package-upgrade/
      SKILL.md
      references/
        epel-environments.md      # per-version CentOS Stream container setup (8/9/10)
        epel-gotchas.md           # request-branch, manual changelog, fedpkg update fallback
```

Plus registry/doc updates: `.claude-plugin/marketplace.json`, root `README.md`, root `CLAUDE.md` table.

## Generalization — placeholders

The runbook hardcodes one maintainer's identity. These become documented placeholders:

| Placeholder | Replaces (original) | Used in |
|:------------|:--------------------|:--------|
| `<FAS_USERNAME>` | `dm4ss` | `fkinit -u <FAS_USERNAME>` |
| `<MAINTAINER_NAME>` | `dm4` | `git config --global user.name` |
| `<MAINTAINER_EMAIL>` | `dm4@secondstate.io` | `git config --global user.email` |
| `<PAGURE_TOKEN>` | `<token>` | `fedpkg set-pagure-token` |
| `<x.y.z>` / `$VERSION` | e.g. `0.13.5` | spec `Version:`, tarball name, branch script |

A short "Fill these in once" note at the top of `environment-setup.md` lists them.

---

## Skill 1 — `fedora-package-upgrade`

### Intent
Guide a maintainer through releasing a new WasmEdge version to Fedora: **Rawhide first**,
then the live **stable (`f4x`) branches**. The assistant explains each stage and hands exact
copy-paste commands; the human performs the credentialed/interactive steps.

### Scope (what it covers)
1. **Environment setup** — privileged `fedora:rawhide` container, `fedora-packager` toolchain,
   kerberos ccache fix, SSH key, `git config`, `fkinit`, `fedpkg co wasmedge`.
2. **Rawhide release** — the canonical per-version loop:
   - install BuildRequires via `grep BuildRequires wasmedge.spec`
   - baseline `fedpkg local` to confirm the env
   - bump `Version:` in `wasmedge.spec`
   - `spectool -g` to fetch the new source tarball
   - `fedpkg upload <tarball>` to the lookaside cache
   - `fedpkg local` to verify the new build
   - `git add -p` → `git commit -s -m 'Release <x.y.z>'` → `fedpkg push`
   - `fedpkg build` (Koji) → **bodhi update auto-created on Rawhide only**
   - download built RPMs: `bodhi updates download --updateid <FEDORA-YYYY-xxxx>`
3. **Stable branches (`f4x`)** — after the Rawhide bodhi update lands:
   - check the EOL page for which branches are still live
   - `fedpkg switch-branch f<NN>` → `git merge rawhide`
   - `mock -r /etc/mock/fedora-<NN>-x86_64.cfg wasmedge-<VERSION>-*.fc<NN>.src.rpm`
   - `fedpkg push` → `fedpkg build` → **manual** `fedpkg update` (type=enhancement)

### Human-only steps (assistant must hand off, not run)
`fkinit`, editing `~/.ssh/id_rsa`, `vim wasmedge.spec`, `git add -p`, `fedpkg push`,
`fedpkg build`, `fedpkg update`, and the interactive `read -p "Ready to push?"` gate.

### Key anti-patterns to encode
- Doing stable branches before Rawhide's bodhi update succeeds.
- Forgetting only Rawhide auto-creates the bodhi update; stable branches need a **manual** `fedpkg update`.
- Running the container without `--privileged` (mock/chroot breaks).
- Skipping the kerberos `default_ccache_name` fix (breaks `fkinit`).
- SSH key without `chmod 400` (breaks `fedpkg push` over SSH).
- Building stable without a `mock` test of the SRPM first.

### References
| File | Contains |
|:-----|:---------|
| `environment-setup.md` | Container run, packager tools, krb5 fix, SSH key, git config, `fkinit`, placeholders |
| `rawhide-release.md` | The canonical per-version loop + bodhi/koji follow-up (also reused by the EPEL skill) |
| `branched-release.md` | The `f4x` branch script, mock test, manual update, EOL check |

---

## Skill 2 — `epel-package-upgrade`

### Intent
Guide a maintainer through building/releasing WasmEdge for **EPEL 8/9/10** *after* the Fedora
release is done. Reuses the Fedora skill's canonical loop and layers on the EPEL-specific deltas.

### Scope (what it covers)
1. **Prerequisite & branch request** — Fedora release complete; if the EPEL branch doesn't exist:
   `fedpkg set-pagure-token <PAGURE_TOKEN>` → `fedpkg request-branch --repo wasmedge epel<N>`.
2. **Per-version container setup** (the main EPEL difference):
   - **epel8** — `quay.io/centos/centos:stream8`: swap mirrorlist→`vault.centos.org`, enable
     `powertools`, install `epel-release epel-next-release glibc-langpack-en`.
   - **epel9** — `quay.io/centos/centos:stream9`: enable `crb`, install `epel-release epel-next-release`.
   - **epel10** — `quay.io/centos/centos:stream10`: enable `crb`, install `epel-release-latest-10.noarch.rpm`.
3. **Run the canonical loop with EPEL deltas:**
   - `%autorelease`/`%autochangelog` **unavailable** → after `git merge rawhide`, hand-edit the
     changelog in `wasmedge.spec`, `git add -p`, `git commit -s -m 'Update changelog'`, then continue.
   - `fedora-review` / `fedora-packager-kerberos` missing on these images is **expected** — ignore.
4. **Ship** — `fedpkg push` → `fedpkg build` → `fedpkg update`; if `fedpkg update` fails, add the
   update directly on the bodhi web UI (bodhi.fedoraproject.org).

### Key anti-patterns to encode
- Using `%autorelease`/`%autochangelog` on EPEL (not supported → write changelog manually).
- Using `powertools` on EPEL 9/10 (it's `crb`) or `crb` on EPEL 8 (it's `powertools`).
- Forgetting the CentOS Stream 8 `vault.centos.org` repo swap (mirrors are EOL/gone).
- Adding `epel-next-release` on epel10 (uses the `epel-release-latest-10` rpm instead).
- Treating missing `fedora-review` as an error on EPEL images.
- Expecting an automatic bodhi update (EPEL always needs a manual `fedpkg update` / web UI).

### References
| File | Contains |
|:-----|:---------|
| `epel-environments.md` | Version→image→repo matrix and exact container setup for stream8/9/10 |
| `epel-gotchas.md` | `request-branch`, manual changelog procedure, `fedpkg update` web-UI fallback |

The EPEL `SKILL.md` explicitly directs the reader to the Fedora skill's `rawhide-release.md`
for the shared core loop rather than restating it.

---

## SKILL.md shape (both skills)

Following the repo's established pattern (`knope-rust`, `vision-verify`):
`name` + trigger-rich `description` frontmatter → `## Role` → `## Workflow` (numbered stages that
point into `references/`) → `## Quick Reference` (command sequence + the release-order matrix) →
`## Anti-Patterns` → `## Reference Guide` (table). Target < 2000 words each; detail lives in references.

## Out of scope (this iteration)
- Other WasmEdge skills (the user noted more will follow) — separate future specs.
- The actual content of `wasmedge.spec` (version-bump mechanics beyond `Version:`/changelog).
- Debian/Ubuntu, Homebrew, or other distro packaging.
- Fully automating credentialed steps (kerberos/SSH/push remain human-run by design).

## Implementation plan (commit after each phase)
1. **Phase 1** — Scaffold `plugins/wasmedge/` (`plugin.json`), register in `marketplace.json`,
   write the `fedora-package-upgrade` skill (SKILL.md + 3 references). Commit + push.
2. **Phase 2** — Write the `epel-package-upgrade` skill (SKILL.md + 2 references). Commit + push.
3. **Phase 3** — Update root `README.md` and `CLAUDE.md` "Current Plugins" table. Commit + push.

(No automated tests exist for markdown skills; verification = `lineguard` on changed files +
a manual read-through against this design.)
