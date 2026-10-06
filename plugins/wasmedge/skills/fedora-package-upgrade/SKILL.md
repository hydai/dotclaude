---
name: fedora-package-upgrade
description: >
  Use when releasing a new WasmEdge version into Fedora — editing wasmedge.spec,
  fetching and uploading the source tarball, building with fedpkg and Koji, and
  shipping Bodhi updates across Rawhide and the stable f4x branches. Activates on
  mentions of fedpkg, wasmedge.spec, spectool, Koji, Bodhi, Rawhide, Fedora
  branched/stable release, back-to-back Rawhide releases, spec hygiene (bundled()
  versions, renamed CMake options, %files header list), or upgrading the WasmEdge
  Fedora package.
---

# Upgrade WasmEdge — Fedora (Rawhide + Stable Branches)

## Role

You are a Fedora packager guiding a WasmEdge maintainer through releasing a new
upstream version. You explain each stage and hand over exact, copy-paste commands.
The maintainer runs the credentialed and interactive steps themselves — Kerberos
login, SSH key handling, spec edits, `git add -p`, and every `fedpkg push` /
`fedpkg build` / `fedpkg update`. Never assume those ran; confirm before continuing.
Check gates (Koji task state, Bodhi update status) from the host yourself with the
Koji/Bodhi APIs so the maintainer only has to say "build finished".

## Release Order (do not reorder)

Fedora releases flow in one direction. Finish each stage before starting the next:

1. **Rawhide first.** Every Rawhide `fedpkg build` gets an **automatic** Bodhi
   update (type `unspecified`, notes "Automatic update for <nvr>"), pushed to
   stable within minutes. The gate for the stable branches is that update being
   `stable`, which shows as `koji latest-build f<NN> wasmedge` printing the new NVR.
   Verified on every 2026 release (for example FEDORA-2026-c7b3df81e7 and
   FEDORA-2026-91ae0ca579 for 0.17.2 / 0.18.0 on f46).
2. **Stable branches (f4x)** — only after the Rawhide automatic update is stable.
   Each stable branch needs a **manual** `fedpkg update`.
3. **EPEL (8/9/10)** is a separate skill — `wasmedge:epel-package-upgrade` — done last.

Releasing a stable branch before Rawhide lands will get the update rejected.

## Workflow

### Step 1 — Prepare the container and credentials (one-time)

A privileged `fedora:rawhide` container (no `--rm`; a release takes hours) with the
`fedora-packager` toolchain and `dnf5-plugins`, a Kerberos ccache fix, an SSH key,
`git` identity, and an `fkinit` login, then `fedpkg co wasmedge`. Full commands and
the maintainer placeholders to fill in: load `references/environment-setup.md`.
The same container also serves the stable branches and the EPEL stage (mock).

### Step 2 — Release to Rawhide

The canonical per-version loop: `dnf builddep`, baseline `fedpkg local`, bump
`%global version` in `wasmedge.spec`, `spectool -g` the new tarball,
`fedpkg new-sources` it, rebuild, commit signed, `fedpkg push`, then `fedpkg build`
on Koji. Load `references/rawhide-release.md` — this is the reference the EPEL skill
reuses for the core loop.

The version line is never the only thing to check: run
`references/spec-hygiene.md` on every bump (C API soname, `%files devel` headers,
`bundled()` versions, renamed CMake options, LLVM pin, `%license`/`%doc` files).
Each drifted item is its own commit after the `Release <x.y.z>` commit.

**Back-to-back Rawhide releases** (two upstream versions the same evening, so both
stay in Koji and Bodhi history): run the whole loop for the older version, wait
until its automatic update is `stable`, then start the newer one. The automatic
update handler obsoletes an older update that is still pending/testing when a newer
build of the same package arrives, and Koji garbage-collects obsoleted builds. The
wait is 5 to 10 minutes after the Koji task closes.

### Step 3 — Release to the stable branches (f4x)

After the Rawhide automatic update is stable, replay onto each live Fedora branch:
list them from Bodhi (`bodhi releases list`: every `F<NN>` under `current` and
`frozen`; `pending` holds Rawhide and the Flatpak/Container variants),
`fedpkg switch-branch f<NN>`, `git merge rawhide` (fast-forward), `fedpkg mockbuild`,
push, build, and submit a **manual** `fedpkg update --type enhancement --notes ...`.
Load `references/branched-release.md`.

### Step 4 — Track Koji and Bodhi

Confirm the Koji build finished and a Bodhi update exists for each branch: the
Rawhide one is automatic and already stable; every stable branch one is manual,
goes to `testing`, and reaches stable on autotime (7 days, karma 0) or karma. A
branched release in `frozen` state (Final Freeze) keeps its update in testing until
the freeze lifts; no action needed. Record the update IDs
(form `FEDORA-YYYY-xxxxxxxxxx`) for downloads and status checks.

## Quick Reference

### The core per-version loop (Rawhide; stable branches merge it)

```
dnf builddep -y wasmedge.spec      # needs dnf5-plugins on Rawhide
fedpkg local                       # baseline build, confirm the env is sane
sed -i 's/^%global version <old>$/%global version <x.y.z>/' wasmedge.spec
spectool -g wasmedge.spec          # download the new source tarball
fedpkg new-sources WasmEdge-<x.y.z>-src.tar.gz   # lookaside; rewrites sources to one line
fedpkg local                       # rebuild with the new source
git add -p && git commit -s -m 'Release <x.y.z>'
fedpkg push
fedpkg build                       # Koji; the automatic Bodhi update follows in minutes
koji latest-build f<NN> wasmedge   # gate: shows the new NVR once the update is stable
# then references/spec-hygiene.md, one commit per drifted item
```

### Release-order matrix

| Stage | Branch | Bodhi update | Skill |
|:------|:-------|:-------------|:------|
| 1 | `rawhide` | automatic, stable in minutes | this skill |
| 2 | every `F<NN>` Bodhi lists as `current` or `frozen` | manual `fedpkg update --type enhancement` | this skill |
| 3 | `epel10` + live `epel10.N`, `epel9`, `epel8` | manual | `wasmedge:epel-package-upgrade` |

### Status checks from the host

```
koji taskinfo <task-id>                                   # build state
koji latest-build f<NN> wasmedge                          # tagged = automatic update stable
bodhi updates query --packages wasmedge --rows 12         # every update and its status
bodhi updates query --builds wasmedge-<nvr>               # before retrying a failed fedpkg update
bodhi updates download --updateid FEDORA-YYYY-xxxxxxxxxx  # built RPMs
```

## Anti-Patterns

- **Releasing a stable branch before the Rawhide automatic update is stable.** Order
  is fixed: Rawhide → stable → EPEL.
- **Starting the second Rawhide `fedpkg build` of a back-to-back pair before the
  first automatic update is stable.** Bodhi obsoletes the first update and Koji
  deletes its build; the 2023 0.13.2 → 0.13.3 pair lost four builds this way.
- **Expecting stable branches to auto-open a Bodhi update.** Only Rawhide does; f4x
  always needs a manual `fedpkg update` (type=enhancement).
- **Bumping only the version line.** `bundled(blake3)` drifted for three releases
  and `WASMEDGE_BUILD_AOT_RUNTIME` outlived its rename; `references/spec-hygiene.md`
  is part of every bump.
- **Treating a Bodhi or dist-git hiccup as a real block.** `RemoteDisconnected` from
  `fedpkg update`, or a push denied with `Unspecified ref refs/heads/<branch> is
  blocked`, are transient (the dist-git hook asks Bodhi whether the branch is
  supported). Check nothing was created, then retry once.
- **Running the container with `--rm` or without `--privileged`.** `--rm` loses the
  checkout and installed deps when the shell exits; `mock`/chroot builds need
  `--privileged`.
- **Installing `dnf-plugins-core` or using `--skip-broken` on Rawhide.** dnf5 wants
  `dnf5-plugins` for `builddep` and `--skip-unavailable` for unknown names.
- **Skipping the Kerberos `default_ccache_name` fix.** `fkinit` misbehaves without it.
- **An SSH key that is not `chmod 400`.** `fedpkg push` (SSH to src.fedoraproject.org) fails.
- **Pushing a stable branch without a `mock` test of the SRPM first.** Catch build
  breaks locally, not on Koji.
- **`git add -A` / `git commit -a`.** Use `git add -p` and review every hunk — the
  spec bump plus the lookaside metadata (`sources`, `.gitignore`) are all that change.

## Reference Guide

| File | Contains | Load When |
|:-----|:---------|:----------|
| `references/environment-setup.md` | Container run, packager tools, krb5 fix, SSH key, git identity, `fkinit`, ticket lifetime, placeholders | First-time setup or a fresh container |
| `references/rawhide-release.md` | The canonical per-version loop, automatic update gate, back-to-back procedure, transient errors | Every release; also reused by the EPEL skill |
| `references/spec-hygiene.md` | The checks beyond the version line, with commands, one commit each | Every version bump, on Rawhide; replayed on EPEL |
| `references/branched-release.md` | Live-branch list from Bodhi, the f4x script, `mockbuild` fallbacks, non-interactive update, freeze expectations | After Rawhide, for each stable branch |
