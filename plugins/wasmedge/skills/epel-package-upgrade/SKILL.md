---
name: epel-package-upgrade
description: >
  Use when building or releasing WasmEdge for EPEL (8, 9, or 10) — replaying the
  Rawhide version bump onto the epel branches, mock-testing against the
  centos-stream+epel chroots, hand-writing the changelog where the branch's spec has
  no %autorelease, shipping a manual Bodhi update, and covering the epel10.N minor
  branches. Activates on mentions of EPEL, epel8/epel9/epel10/epel10.N, CentOS
  Stream, powertools, crb, request-branch, %autorelease on RHEL, RHEL buildroot
  LLVM, or the WasmEdge EPEL build.
---

# Upgrade WasmEdge — EPEL (8 / 9 / 10)

## Role

You are an EPEL packager guiding a WasmEdge maintainer through building for
RHEL/CentOS via EPEL. EPEL reuses Fedora's core release loop, so this skill covers
only the EPEL-specific differences: where to build, the diverged specs, the
branch-dependent changelog, the always-manual Bodhi update, and the EPEL 10 minor
branches. You hand over exact commands; the maintainer runs the credentialed and
interactive steps.

## Prerequisite

Do EPEL **last**, after the Fedora Rawhide and stable-branch releases are done
(see `wasmedge:fedora-package-upgrade`). EPEL branches do **not** merge from
`rawhide` (they diverged: different LLVM, `ExclusiveArch`, changelog style); they
replay the Rawhide bump by hand. Only `epel10.N` merges, from `epel10`.

## Workflow

### Step 1 — Request the EPEL branch (one-time, if missing)

If the `epel<N>` branch does not exist yet, request it through Pagure:

```
fedpkg set-pagure-token <PAGURE_TOKEN>
fedpkg request-branch --repo wasmedge epel<N>     # epel8 | epel9 | epel10
```

### Step 2 — Build from the Fedora rawhide container (default)

Stay in the `wasmedge-rawhide` container from the Fedora skill: it has the
checkout, the Kerberos ticket, a current `bodhi-client`, and `mock-core-configs`
with the EPEL chroots. Local tests are `fedpkg mockbuild`, not `fedpkg local`:

| Branch | mock config | Command |
|:-------|:------------|:--------|
| `epel10` | `epel-10-<arch>` (CentOS Stream 10 + EPEL, what Koji's `epel10.N-build` uses) | `fedpkg mockbuild` (auto-selected) |
| `epel9` | `centos-stream+epel-9-<arch>` (no `epel-9-<arch>.cfg` exists) | `fedpkg mockbuild --root centos-stream+epel-9-<arch>` |
| `epel8` | `centos-stream+epel-8-<arch>` — **stale**, see Step 4 | `fedpkg mockbuild --root centos-stream+epel-8-<arch>` |
| `epel10.N` | none needed; same spec as `epel10` | skip, build on Koji |

`<arch>` is the host arch (`aarch64` on Apple Silicon). The per-version CentOS
Stream containers in `references/epel-environments.md` are the fallback when mock
is unavailable; they bring their own traps (crb persistence, dnf ordering, EOL
Stream 8 mirrors).

### Step 3 — Replay the bump and the changelog

**Check the branch's spec first** — the style differs per branch and is not what
"EPEL supports rpmautospec" suggests:

```
fedpkg switch-branch epel<N>
grep -nE 'autorelease|autochangelog|^Release:' wasmedge.spec
```

- `Release: %autorelease` + `%autochangelog` (epel10, epel10.N): the commit message
  is the changelog; commit as `Release <x.y.z>`.
- Literal `Release: 1%{?dist}` + hand-maintained `%changelog` (epel9 **and** epel8
  as of 0.18.0): add a dated entry; commit as `Update to <x.y.z>`.

Then replay the Rawhide bump. The tarball is already in the lookaside (it is per
package), so copy the metadata instead of re-uploading:

```
sed -i 's/^%global version <old>$/%global version <x.y.z>/' wasmedge.spec
# manual-changelog branches only (weekday must match the date):
sed -i "/^%changelog$/a * $(date +'%a %b %d %Y') <MAINTAINER_NAME> <<MAINTAINER_EMAIL>> - <x.y.z>-1\n- Update to <x.y.z>\n" wasmedge.spec
git checkout rawhide -- sources .gitignore
diff <(git show rawhide:wasmedge.spec) wasmedge.spec     # reconcile; see gotchas
```

Reconcile the body against rawhide: **keep** the platform lines (LLVM major in
`BuildRequires`/`Requires`, `-DLLVM_DIR=`, `ExclusiveArch`, `gcc-toolset` lines,
legacy `License:` string), **port** everything else, including the
`references/spec-hygiene.md` commits Rawhide made this release. Then
`fedpkg mockbuild` per Step 2, `git add -p`, commit signed, `fedpkg push`.

### Step 4 — Build and ship a manual update

```
fedpkg build
fedpkg verrel                                          # wasmedge-<x.y.z>-1.el<N>
fedpkg update --type enhancement --notes 'Update to <x.y.z>'
```

EPEL never auto-opens a Bodhi update. `fedpkg update` works for **every** EPEL
branch from the rawhide container; the "web UI mandatory on EPEL 8" rule only
applies when running a Stream 8 container whose `bodhi-client < 6.0.0` rejects the
command. Transient failures (`RemoteDisconnected`, a push denied with
`Unspecified ref refs/heads/epel<N> is blocked`): confirm nothing was created
(`bodhi updates query --builds <nvr>`), then retry once.

**EPEL 8 mock is stale.** The `centos-stream+epel-8` chroot is the EOL CentOS
Stream 8 snapshot (LLVM 17.0.6), but Koji builds epel8 against RHEL 8.10, which
carried LLVM 21 in 2026. A Stream 8 mock failure that is an LLVM API error
(`LLVMSetTailCallKind` not declared, 0.18.0) is a false negative: push and build
on Koji. Check the real buildroot with the previous build:
`koji rpminfo wasmedge-<prev>-1.el8.x86_64 | grep Buildroot` then
`koji list-buildroot <id> | grep -E '^(llvm|lld|gcc-toolset)'`.

Load `references/epel-gotchas.md` for the full list.

### Step 5 — EPEL 10: also ship the minor-pinned `epel10.N` branches

EPEL 10 uses a **rolling** `epel10` branch (builds against CentOS Stream 10, dist
tag `.el10_<next minor>`) **plus** minor-pinned `epel10.N` branches (`epel10.2`,
`epel10.3` in October 2026) for users on an older, still-supported RHEL 10 minor.
Ship to `epel10` and to every `epel10.N` that Bodhi lists as `current`
(`bodhi releases list`). The dist tag is set by `%autorelease` per branch.

Minor branches merge from `epel10` (**not** rawhide) — a fast-forward once
`epel10` is pushed:

```
fedpkg switch-branch epel10.N
git merge epel10
fedpkg push && fedpkg build && fedpkg verrel
fedpkg update --type enhancement --notes 'Release <x.y.z>'
```

## Quick Reference

### Branch → buildroot → spec style (wasmedge, 2026)

| Branch | Koji buildroot | Spec style | LLVM |
|:-------|:---------------|:-----------|:-----|
| `epel10` | CentOS Stream 10 (`epel10.N-build`, N = next minor) | `%autorelease` | `llvm19` compat from EPEL 10 itself |
| `epel10.N` | RHEL 10.N / Stream snapshot | `%autorelease` | same |
| `epel9` | RHEL 9 | literal Release + manual changelog | system `llvm-devel` |
| `epel8` | RHEL 8.10 | literal Release + manual changelog + `gcc-toolset-11` | system `llvm-devel` (21 in 2026) |

### EPEL deltas vs the Fedora loop

- No `git merge rawhide`: version sed + `git checkout rawhide -- sources .gitignore`
  + reconcile diff.
- Changelog per branch style (above). `%autochangelog` reads the commit message.
- Bodhi update is **always manual** (`fedpkg update --type enhancement --notes`).
- Local test is mock from the rawhide container; Stream 8 results are advisory.
- `fedora-review` / `fedora-packager-kerberos` absent on Stream images — expected.

## Anti-Patterns

- **`git merge rawhide` on an EPEL branch.** Conflicts in all three files
  (verified 2026-10-06 on every EPEL branch); one slip on an LLVM line → unbuildable.
- **Assuming epel9 uses `%autorelease` because EPEL 9 supports it.** The wasmedge
  epel9 spec is literal `Release:` + hand-written changelog, like epel8. grep first.
- **Trusting a Stream 8 mock failure.** The chroot is two years behind RHEL 8.10.
- **Starting CentOS Stream containers by default.** mock from the rawhide container
  covers epel9/epel10 faithfully and needs no extra setup.
- **Bundling `epel-release` with an EPEL-only package** (`rpmautospec`,
  `fedora-packager`) in one `dnf install` when you do use a Stream container. `dnf`
  resolves every argument against the *currently* enabled repos, so it fails with
  `No match for argument: rpmautospec` and installs **nothing, including
  `epel-release`**. Enable EPEL in its own transaction first.
- **`crb` on EPEL 8, or `powertools` on EPEL 9/10.** Stream 8 uses `powertools`;
  Stream 9 and 10 use `crb`.
- **Skipping the `vault.centos.org` swap on Stream 8.** CentOS 8 mirrors are EOL.
- **Adding `epel-next-release` on EPEL 10.** Use the `epel-release-latest-10` RPM.
- **Waiting for an automatic Bodhi update.** EPEL always needs a manual one.
- **Reading a big fmt-6 / spdlog error count as "EPEL 8 can't take this release."**
  Count ≠ difficulty — mechanical breakage (const-ness, formatter context type,
  missing `detail` symbols) is guardable, and 0.17.0 shipped to EPEL 8 this way;
  since 0.17.1 the guards are upstream and no patch is needed.
  Triage the *kinds* of error first; see `references/epel-gotchas.md`.

## Reference Guide

| File | Contains | Load When |
|:-----|:---------|:----------|
| `references/epel-gotchas.md` | Changelog per branch, spec reconciliation, mock from the rawhide container and the Stream 8 staleness, LLVM floor of a release, transient Bodhi/dist-git errors, OIDC localhost-callback fix, fmt/spdlog compat history | Releasing on any EPEL branch |
| `references/epel-environments.md` | Fallback per-version CentOS Stream container + repo setup (8/9/10) | Only when mock cannot be used |
| `wasmedge:fedora-package-upgrade` → `references/spec-hygiene.md` | The checks beyond the version line | Replaying the Rawhide commits onto an EPEL spec |

> The shared "bump → build → ship" loop lives in the Fedora skill:
> `wasmedge:fedora-package-upgrade` → `references/rawhide-release.md`.
