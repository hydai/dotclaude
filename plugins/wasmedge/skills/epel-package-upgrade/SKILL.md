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
the manual changelog, and the always-manual Bodhi update. You hand over exact
commands; the maintainer runs the credentialed and interactive steps.

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

### Step 3 — Merge Rawhide and hand-write the changelog

EPEL does not support `%autorelease`/`%autochangelog`, so the changelog is manual:

```
fedpkg switch-branch epel<N>
git merge rawhide
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
fails, add the update on the Bodhi web UI. Load `references/epel-gotchas.md`.

## Quick Reference

### Version → image → repo matrix

| EPEL | Base image | Enable repo | EPEL release packages |
|:-----|:-----------|:------------|:----------------------|
| epel8 | `quay.io/centos/centos:stream8` | `powertools` (+ `vault.centos.org` swap) | `epel-release epel-next-release glibc-langpack-en` |
| epel9 | `quay.io/centos/centos:stream9` | `crb` | `epel-release epel-next-release` |
| epel10 | `quay.io/centos/centos:stream10` | `crb` | `epel-release-latest-10.noarch.rpm` (from a URL) |

### EPEL deltas vs the Fedora loop

- Changelog is **manual** (no `%autorelease`).
- Bodhi update is **always manual** (`fedpkg update`, or the web UI).
- `fedora-review` / `fedora-packager-kerberos` absent — expected, ignore.

## Anti-Patterns

- **Using `%autorelease`/`%autochangelog` on EPEL.** Unsupported — write the
  changelog by hand after `git merge rawhide`.
- **`crb` on EPEL 8, or `powertools` on EPEL 9/10.** Stream 8 uses `powertools`;
  Stream 9 and 10 use `crb`.
- **Skipping the `vault.centos.org` swap on Stream 8.** CentOS 8 mirrors are EOL —
  the default `mirrorlist` URLs are dead.
- **Adding `epel-next-release` on EPEL 10.** Use the `epel-release-latest-10` RPM.
- **Treating missing `fedora-review` as a failure.** It is not packaged for these
  images; you do not need it.
- **Waiting for an automatic Bodhi update.** EPEL always needs a manual one.

## Reference Guide

| File | Contains | Load When |
|:-----|:---------|:----------|
| `references/epel-environments.md` | Per-version CentOS Stream container + repo setup (8/9/10) | Bringing up a build container |
| `references/epel-gotchas.md` | `request-branch`, manual changelog procedure, `fedpkg update` web-UI fallback | Releasing on any EPEL branch |

> The shared "bump → build → ship" loop lives in the Fedora skill:
> `wasmedge:fedora-package-upgrade` → `references/rawhide-release.md`.
