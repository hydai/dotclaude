---
name: fedora-package-upgrade
description: >
  Use when releasing a new WasmEdge version into Fedora — editing wasmedge.spec,
  fetching and uploading the source tarball, building with fedpkg and Koji, and
  shipping Bodhi updates across Rawhide and the stable f4x branches. Activates on
  mentions of fedpkg, wasmedge.spec, spectool, Koji, Bodhi, Rawhide, Fedora
  branched/stable release, or upgrading the WasmEdge Fedora package.
---

# Upgrade WasmEdge — Fedora (Rawhide + Stable Branches)

## Role

You are a Fedora packager guiding a WasmEdge maintainer through releasing a new
upstream version. You explain each stage and hand over exact, copy-paste commands.
The maintainer runs the credentialed and interactive steps themselves — Kerberos
login, SSH key handling, `vim` spec edits, `git add -p`, and every `fedpkg push` /
`fedpkg build` / `fedpkg update`. Never assume those ran; confirm before continuing.

## Release Order (do not reorder)

Fedora releases flow in one direction. Finish each stage before starting the next:

1. **Rawhide first.** Only Rawhide auto-creates a Bodhi update after `fedpkg build`.
2. **Stable branches (f4x)** — only after the Rawhide Bodhi update is submitted.
   Each stable branch needs a **manual** `fedpkg update`.
3. **EPEL (8/9/10)** is a separate skill — `wasmedge:epel-package-upgrade` — done last.

Releasing a stable branch before Rawhide lands will get the update rejected.

## Workflow

### Step 1 — Prepare the container and credentials (one-time)

A privileged `fedora:rawhide` container with the `fedora-packager` toolchain, a
Kerberos ccache fix, an SSH key, `git` identity, and an `fkinit` login, then
`fedpkg co wasmedge`. Full commands and the maintainer placeholders to fill in:
load `references/environment-setup.md`.

### Step 2 — Release to Rawhide

The canonical per-version loop: install BuildRequires, baseline `fedpkg local`,
bump `Version:` in `wasmedge.spec`, `spectool -g` the new tarball, `fedpkg upload`
it, rebuild, commit signed, `fedpkg push`, then `fedpkg build` on Koji. The Bodhi
update is created automatically; download built RPMs with `bodhi updates download`.
Load `references/rawhide-release.md` — this is the reference both this skill and
the EPEL skill reuse for the core loop.

### Step 3 — Release to the stable branches (f4x)

After the Rawhide Bodhi update is in, replay onto each live Fedora branch: check
the EOL list, `fedpkg switch-branch f<NN>`, `git merge rawhide`, `mock`-test the
SRPM, push, build, and submit a **manual** `fedpkg update` (type=enhancement).
Load `references/branched-release.md`.

### Step 4 — Track Koji and Bodhi

Confirm the Koji build finished and a Bodhi update exists for each branch.
Rawhide's is automatic; every stable branch is manual. Record the Bodhi update
IDs (form `FEDORA-YYYY-xxxxxxxxxx`) for downloads and status checks.

## Quick Reference

### The core per-version loop (Rawhide and every branch)

```
dnf install -y $(grep BuildRequires wasmedge.spec | cut -d' ' -f2)  # build deps
fedpkg local                       # baseline build, confirm the env is sane
vim wasmedge.spec                  # bump Version: (%autorelease handles the rest on Fedora)
spectool -g wasmedge.spec          # download the new source tarball
fedpkg upload WasmEdge-<x.y.z>-src.tar.gz   # push tarball to the lookaside cache
fedpkg local                       # rebuild with the new source
git add -p && git commit -s -m 'Release <x.y.z>'
fedpkg push
fedpkg build                       # Koji build (Rawhide auto-opens a Bodhi update)
```

### Release-order matrix

| Stage | Branch | Bodhi update | Skill |
|:------|:-------|:-------------|:------|
| 1 | `rawhide` | automatic | this skill |
| 2 | `f41`, `f40`, … (live only) | manual `fedpkg update` | this skill |
| 3 | `epel10`, `epel9`, `epel8` | manual | `wasmedge:epel-package-upgrade` |

### Download a built update

```
bodhi updates download --updateid FEDORA-YYYY-xxxxxxxxxx
```

## Anti-Patterns

- **Releasing a stable branch before Rawhide's Bodhi update is in.** Order is fixed:
  Rawhide → stable → EPEL.
- **Expecting stable branches to auto-open a Bodhi update.** Only Rawhide does; f4x
  always needs a manual `fedpkg update` (type=enhancement).
- **Running the container without `--privileged`.** `mock`/chroot builds fail.
- **Skipping the Kerberos `default_ccache_name` fix.** `fkinit` misbehaves without it.
- **An SSH key that is not `chmod 400`.** `fedpkg push` (SSH to src.fedoraproject.org) fails.
- **Pushing a stable branch without a `mock` test of the SRPM first.** Catch build
  breaks locally, not on Koji.
- **`git add -A` / `git commit -a`.** Use `git add -p` and review every hunk — the
  spec bump plus the lookaside metadata (`sources`, `.gitignore`) are all that change.

## Reference Guide

| File | Contains | Load When |
|:-----|:---------|:----------|
| `references/environment-setup.md` | Container run, packager tools, krb5 fix, SSH key, git identity, `fkinit`, placeholders | First-time setup or a fresh container |
| `references/rawhide-release.md` | The canonical per-version loop + Koji/Bodhi follow-up | Every release; also reused by the EPEL skill |
| `references/branched-release.md` | The f4x stable-branch script, `mock` test, manual update, EOL check | After Rawhide, for each stable branch |
