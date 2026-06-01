# Rawhide Release — The Canonical Per-Version Loop

This is the core "bump → build → ship" loop. The stable-branch and EPEL flows
reuse it; they only change the environment and the changelog/update handling.

Run from inside the checked-out `wasmedge` repo on the `rawhide` branch.

## 1. Install build dependencies

```
dnf install -y $(grep BuildRequires wasmedge.spec | cut -d' ' -f2)
```

## 2. Baseline build (confirm the env)

Build the current version once before changing anything, so a later failure is
clearly from your change and not the environment:

```
fedpkg local
```

## 3. Bump the version (human step — vim)

Edit `wasmedge.spec`:
- update `Version:` to the new `<x.y.z>`

On Fedora the spec uses `%autorelease` and `%autochangelog`, so `Release:` and the
changelog are generated automatically — `Version:` is usually the only line you
change. (EPEL is different: `%autorelease` is unavailable there and the changelog
is written by hand. See the `wasmedge:epel-package-upgrade` skill.)

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
