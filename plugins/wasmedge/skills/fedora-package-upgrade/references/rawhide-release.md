# Rawhide Release — The Canonical Per-Version Loop

This is the core "bump → build → ship" loop. The stable-branch and EPEL flows
reuse it; they only change the environment and the changelog/update handling.

Run from inside the checked-out `wasmedge` repo on the `rawhide` branch.

## 1. Install build dependencies

Use `dnf builddep` — it parses `BuildRequires` correctly, including version
constraints and multi-package lines. On Rawhide (dnf5) it comes from
`dnf5-plugins`; `dnf-plugins-core` is the dnf4 package and does not provide it:

```
dnf builddep -y wasmedge.spec
```

Checklist — pinned LLVM major: the spec pins an LLVM major in `BuildRequires`
(e.g. `llvm18-devel`, `lld18-devel`). Every upgrade, verify (a) the new upstream
release still supports that LLVM major (`lib/llvm/llvm.h` and `cmake/Helper.cmake`
`LLVM_VERSION_MAJOR` guards), and (b) Fedora rawhide still ships that
`llvmNN-devel` compat package. If `dnf builddep` can't find `llvmNN-devel`,
rawhide retired it → bump the LLVM major in `BuildRequires`.

## 2. Baseline build (confirm the env)

Build the current version once before changing anything, so a later failure is
clearly from your change and not the environment:

```
fedpkg local
```

If the baseline fails on a fresh Rawhide toolchain (nothing may have built the old
version with it yet), that is an environment signal, not a stop: continue with the
new version and judge its own `fedpkg local`.

## 3. Bump the version

Edit `wasmedge.spec`:
- `Version:` may be a macro (e.g. `Version: %{version}`). Find the real definition
  first — `grep -nE '^%global|^Version:' wasmedge.spec` — and bump the
  `%global version` line, not the `Version:` tag (for wasmedge the knob is
  `%global version`):

```
sed -i 's/^%global version <old>$/%global version <x.y.z>/' wasmedge.spec
grep -nE '^%global version|^Version:' wasmedge.spec
```

On Fedora the spec uses `%autorelease` and `%autochangelog`, so `Release:` and the
changelog are generated automatically; the commit message becomes the changelog
entry. The version line is the only line in the **release commit** — but it is
never the only thing to check. Run `spec-hygiene.md` now (C API soname, `%files
devel` headers, `bundled()` versions, renamed CMake options, `%license`/`%doc`
files, upstream changelog scan) and commit each drifted item separately after the
release commit. The classic miss: `WASMEDGE_CAPI_VERSION` / `WASMEDGE_CAPI_SOVERSION`
changed upstream and `%install` fails with `mv: cannot stat '.../libwasmedge.so.<old>'`.
(EPEL is different: `%autorelease` is unavailable on some branches and the changelog
is written by hand. See the `wasmedge:epel-package-upgrade` skill.)

## 4. Download the new source tarball

`spectool` reads the `Source` URL from the spec and downloads it:

```
spectool -g wasmedge.spec
```

## 5. Upload the tarball to the lookaside cache

Fedora does not store tarballs in git — they live in the lookaside cache, which is
**per package, not per branch**: one upload serves rawhide, every f4x branch, and
every EPEL branch.

```
fedpkg new-sources WasmEdge-<x.y.z>-src.tar.gz
cat sources            # exactly one line: SHA512 (WasmEdge-<x.y.z>-src.tar.gz) = ...
```

`fedpkg new-sources` rewrites `sources` to the new tarball only (old tarballs stay
in the lookaside, nothing is lost) and appends to `.gitignore`, so `git add -p`
shows three files. `fedpkg upload` would *append* to `sources` instead; the
wasmedge `sources` carried 16 stale entries until 0.17.2 because of that.

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

`fedpkg push` is plain git over SSH; it needs the SSH key, not a browser. If the
server answers `Unspecified ref refs/heads/<branch> is blocked` / `Denied push for
ref`, the dist-git pre-receive hook could not reach Bodhi to confirm the branch is
supported and fell through to its blacklist. Retry; it is transient (epel9,
2026-10-06). It is a real block only if `bodhi releases list` no longer lists the
branch's release as current/frozen/pending, or the branch has `dead.package`.

## 8. Build on Koji

```
fedpkg build
```

Every Rawhide build gets an **automatic** Bodhi update (type `unspecified`, notes
"Automatic update for <nvr>", no email). It goes pending → testing → stable within
about five minutes of the Koji task closing, and the build is then tagged into the
Rawhide tag. Check from the host:

```
koji latest-build f<NN> wasmedge                 # the gate: prints the new NVR when stable
bodhi updates query --builds wasmedge-<x.y.z>-1.fc<NN>   # status=stable
```

Not stable 30 minutes after the task closed? Look at the update's page; do not
proceed on faith.

**Back-to-back releases (two versions the same evening):** the automatic-update
handler calls Bodhi's `obsolete_older_updates` when a newer build of the same
package arrives. An older update still pending/testing is obsoleted, and Koji's
garbage collection later deletes the obsoleted build (status `DELETED`, as happened
to the 0.13.2 builds in 2023). So: finish the older version's loop, wait for the
gate above, then start the newer version. Everything up to the newer version's
`fedpkg push` can be prepared while waiting; only its `fedpkg build` must wait.

## 9. Download built RPMs (optional)

```
bodhi updates download --updateid FEDORA-YYYY-xxxxxxxxxx
```

## Next

Once the Rawhide automatic update is stable, move to `branched-release.md` for the
stable f4x branches.
