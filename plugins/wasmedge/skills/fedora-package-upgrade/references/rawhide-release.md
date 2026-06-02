# Rawhide Release — The Canonical Per-Version Loop

This is the core "bump → build → ship" loop. The stable-branch and EPEL flows
reuse it; they only change the environment and the changelog/update handling.

Run from inside the checked-out `wasmedge` repo on the `rawhide` branch.

## 1. Install build dependencies

Use `dnf builddep` (from `dnf-plugins-core`) — it parses `BuildRequires`
correctly, including version constraints and multi-package lines:

```
dnf builddep wasmedge.spec
```

Fallback only (fragile — breaks on `cmake >= 3.x` constraints or multiple
packages per line):

```
dnf install -y $(grep BuildRequires wasmedge.spec | cut -d' ' -f2)
```

Checklist — pinned LLVM major: the spec pins an LLVM major in `BuildRequires`
(e.g. `llvm18-devel`, `lld18-devel`). Every upgrade, verify (a) the new upstream
release still supports that LLVM major, and (b) Fedora rawhide still ships that
`llvmNN-devel` compat package. If `dnf builddep` can't find `llvmNN-devel`,
rawhide retired it → bump the LLVM major in `BuildRequires`.

## 2. Baseline build (confirm the env)

Build the current version once before changing anything, so a later failure is
clearly from your change and not the environment:

```
fedpkg local
```

## 3. Bump the version (human step — vim)

Edit `wasmedge.spec`:
- `Version:` may be a macro (e.g. `Version: %{version}`). Find the real definition
  first — `grep -nE '^%global|^Version:' wasmedge.spec` — and bump the
  `%global version` line, not the `Version:` tag (for wasmedge the knob is
  `%global version`).

On Fedora the spec uses `%autorelease` and `%autochangelog`, so `Release:` and the
changelog are generated automatically. The version is usually the only line you
change — but **also check upstream `CMakeLists.txt` for `WASMEDGE_CAPI_VERSION` /
`WASMEDGE_CAPI_SOVERSION`**. If either changed, bump `%global capi_version` /
`%global capi_soname` to match. **Symptom of a miss:** `%install` fails with
`mv: cannot stat '.../libwasmedge.so.<old>'`. (EPEL is different: `%autorelease` is
unavailable there and the changelog is written by hand. See the
`wasmedge:epel-package-upgrade` skill.)

## 4. Download the new source tarball

`spectool` reads the `Source` URL from the spec and downloads it:

```
spectool -g wasmedge.spec
```

## 5. Upload the tarball to the lookaside cache

Fedora does not store tarballs in git — they live in the lookaside cache. This
updates the `sources` file and `.gitignore`:

```
fedpkg upload WasmEdge-<x.y.z>-src.tar.gz
```

`fedpkg upload` *appends* to `sources`. For a version bump `sources` must contain
**only** the new tarball — verify with `cat sources`. If the old
`WasmEdge-<old>-src.tar.gz` entry lingers, overwrite the whole file cleanly with
`fedpkg new-sources WasmEdge-<x.y.z>-src.tar.gz`.

## Patch review (before the rebuild)

Downstream backports graduate upstream and must be dropped on the upgrade that
contains them — otherwise they fail to apply at `%prep`. For each `PatchN` in the
spec, find its upstream commit and test it against the new tag (in an upstream
checkout / worktree):

```
git merge-base --is-ancestor <patch_upstream_commit> <newtag> && echo DROP || echo KEEP
```

If it is already upstream (`DROP`), remove the `PatchN:` line and its comment, then
`git rm` the `.patch` file. `%prep` uses `%autosetup -p1`, which auto-applies all
patches, so dropping the declaration is enough — there is no `%patchN` to remove.
(If a spec uses explicit `%patchN`, remove that line too.)

## 6. Rebuild with the new source

```
fedpkg local
```

Fix any build break before continuing.

## 7. Review, commit, push (human steps)

```
git branch                 # confirm you are on rawhide
git status
git add -p                 # stage hunk by hunk: spec + sources + .gitignore
git commit -s -m 'Release <x.y.z>'
fedpkg push
```

## 8. Build on Koji

```
fedpkg build
```

On **Rawhide only**, a successful Koji build automatically opens a Bodhi update.
The Bodhi link arrives by email (form `FEDORA-YYYY-xxxxxxxxxx`).

## 9. Download built RPMs (optional)

```
bodhi updates download --updateid FEDORA-YYYY-xxxxxxxxxx
```

## Next

Once the Rawhide Bodhi update is submitted, move to `branched-release.md` for the
stable f4x branches.
