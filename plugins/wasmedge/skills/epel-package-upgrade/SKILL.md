---
name: epel-package-upgrade
description: >
  Use when building or releasing WasmEdge for EPEL (8, 9, or 10) on CentOS Stream
  — requesting the EPEL branch, setting up the powertools/crb repos, hand-writing
  the changelog where %autorelease is unavailable, and shipping a manual Bodhi
  update. Activates on mentions of EPEL, epel8/epel9/epel10, CentOS Stream,
  powertools, crb, request-branch, %autorelease on RHEL, or the WasmEdge EPEL build.
---

# Upgrade WasmEdge — EPEL (8 / 9 / 10)

## Role

You are an EPEL packager guiding a WasmEdge maintainer through building for
RHEL/CentOS via EPEL. EPEL reuses Fedora's core release loop, so this skill covers
only the EPEL-specific differences: the CentOS Stream containers, repo enabling,
the branch-dependent changelog, and the always-manual Bodhi update. You hand over
exact commands; the maintainer runs the credentialed and interactive steps.

## Prerequisite

Do EPEL **last**, after the Fedora Rawhide and stable-branch releases are done
(see `wasmedge:fedora-package-upgrade`). EPEL branches merge from `rawhide`.

## Workflow

### Step 1 — Request the EPEL branch (one-time, if missing)

If the `epel<N>` branch does not exist yet, request it through Pagure:

```
fedpkg set-pagure-token <PAGURE_TOKEN>
fedpkg request-branch --repo wasmedge epel<N>     # epel8 | epel9 | epel10
```

### Step 2 — Bring up the right CentOS Stream container

Each EPEL version has a different base image and repo setup — Stream 8 needs the
`vault.centos.org` swap and `powertools`; Stream 9/10 use `crb`; Stream 10 installs
EPEL from an RPM URL. Load `references/epel-environments.md` for the exact per-version
commands. Then do the Fedora maintainer setup (git identity, SSH key, `fkinit`,
`fedpkg co wasmedge`) — `fedora-review` and `fedora-packager-kerberos` are missing
on these images and that is expected.

### Step 3 — Merge Rawhide and update the changelog

**Check the branch's existing spec first.** EPEL 9/10 **do** support
`%autorelease`/`%autochangelog` — if the branch's spec uses them, treat it like a
Fedora stable branch: the commit message *is* the changelog, so do not hand-write a
`%changelog` entry:

```
fedpkg switch-branch epel<N>
git merge rawhide
git add -p
git commit -s -m 'Update to <version>'
```

Only fall back to a hand-written changelog when the branch's spec does **not** use
`%autorelease`/`%autochangelog` (older epel8):

```
vim wasmedge.spec          # add a dated %changelog entry by hand (human step)
git add -p
git commit -s -m 'Update changelog'
```

Then run the canonical build loop from the `wasmedge:fedora-package-upgrade` skill
(`references/rawhide-release.md`), picking up at the local build (`fedpkg local`).
The merge already brought the version bump and lookaside metadata, so you do not
re-edit `Version:` or re-upload the tarball.

### Step 4 — Build and ship a manual update

Optionally `mock`-test the SRPM, then `fedpkg push`, `fedpkg build`, then a
**manual** `fedpkg update`. EPEL never auto-opens a Bodhi update. If `fedpkg update`
fails — **always on EPEL 8**, whose `bodhi-client < 6.0.0` rejects it — add the
update on the Bodhi web UI. (Containerized `fedpkg push` may also hang on an OIDC
localhost callback — see the gotchas.) Load `references/epel-gotchas.md`.

### Step 5 — EPEL 10: also ship the minor-pinned `epel10.N` branches

EPEL 10 uses a **rolling** `epel10` branch (builds against the latest RHEL 10 minor)
**plus** minor-pinned `epel10.N` branches (e.g. `epel10.2`) for users on an
older-but-still-supported RHEL 10 minor. A full EPEL 10 release may need **both**
`epel10` and each live `epel10.N`. The dist tag (`.el10_3` vs `.el10_2`) is auto-set by
`%autorelease` per branch, so you do not touch it.

To bring a new version to a minor branch, merge from `epel10` (**not** rawhide) — same
RHEL 10 LLVM family, so it is usually a clean (often fast-forward) merge, then build and
ship as in Step 4:

```
fedpkg switch-branch epel10.N
git merge epel10
fedpkg push && fedpkg build && fedpkg update
```

## Quick Reference

### Version → image → repo matrix

| EPEL | Base image | Enable repo | EPEL release packages |
|:-----|:-----------|:------------|:----------------------|
| epel8 | `quay.io/centos/centos:stream8` | `powertools` (+ `vault.centos.org` swap) | `epel-release epel-next-release glibc-langpack-en` |
| epel9 | `quay.io/centos/centos:stream9` | `crb` | `epel-release epel-next-release` |
| epel10 | `quay.io/centos/centos:stream10` | `crb` | `epel-release-latest-10.noarch.rpm` (from a URL) |

> EPEL 10 ships as a rolling `epel10` branch **plus** minor-pinned `epel10.N` branches
> (e.g. `epel10.2`). Release to `epel10` and each live `epel10.N` (Step 5); the `.el10_N`
> dist tag is auto-set per branch.

### EPEL deltas vs the Fedora loop

- Changelog depends on the branch's spec: EPEL 9/10 use `%autorelease`/`%autochangelog`
  (commit message = changelog, like a Fedora stable branch); only older epel8 may need a
  **manual** `%changelog`. Check the spec first.
- Bodhi update is **always manual** (`fedpkg update`, or the web UI). On **EPEL 8**
  the web UI is **mandatory** — its `bodhi-client < 6.0.0` rejects `fedpkg update`.
- `fedora-review` / `fedora-packager-kerberos` absent — expected, ignore.

## Anti-Patterns

- **Assuming all EPEL branches need a manual changelog.** EPEL 9/10 support
  `%autorelease`/`%autochangelog` — there the commit message *is* the changelog. Only
  a branch whose spec does not use them (older epel8) needs a hand-written `%changelog`.
  Check the branch's spec before deciding.
- **`crb` on EPEL 8, or `powertools` on EPEL 9/10.** Stream 8 uses `powertools`;
  Stream 9 and 10 use `crb`.
- **Bundling `epel-release` with an EPEL-only package** (`rpmautospec`,
  `fedora-packager`) in one `dnf install`. `dnf` resolves every argument against the
  *currently* enabled repos before installing anything, and EPEL is not enabled yet —
  so it fails with `No match for argument: rpmautospec` and installs **nothing,
  including `epel-release`**. Enable EPEL in its own transaction first
  (`dnf install -y epel-release epel-next-release`), then install the EPEL-only tools.
  Bites on EPEL 9 setup; EPEL 10 already splits the two.
- **Skipping the `vault.centos.org` swap on Stream 8.** CentOS 8 mirrors are EOL —
  the default `mirrorlist` URLs are dead.
- **Adding `epel-next-release` on EPEL 10.** Use the `epel-release-latest-10` RPM.
- **Treating missing `fedora-review` as a failure.** It is not packaged for these
  images; you do not need it.
- **Waiting for an automatic Bodhi update.** EPEL always needs a manual one.
- **Reading a big fmt-6 / spdlog error count as "EPEL 8 can't take this release."**
  Count ≠ difficulty — mechanical breakage (const-ness, formatter context type,
  missing `detail` symbols) is guardable, and 0.17.0 shipped to EPEL 8 this way.
  Triage the *kinds* of error first; see `references/epel-gotchas.md`.

## Reference Guide

| File | Contains | Load When |
|:-----|:---------|:----------|
| `references/epel-environments.md` | Per-version CentOS Stream container + repo setup (8/9/10) | Bringing up a build container |
| `references/epel-gotchas.md` | `request-branch`, changelog, spec/`%files` reconciliation, fmt/spdlog compat patches (incl. the EPEL 8 fmt-6 rescue), Bodhi web UI, OIDC localhost-callback fix | Releasing on any EPEL branch; dependency-version walls |

> The shared "bump → build → ship" loop lives in the Fedora skill:
> `wasmedge:fedora-package-upgrade` → `references/rawhide-release.md`.
