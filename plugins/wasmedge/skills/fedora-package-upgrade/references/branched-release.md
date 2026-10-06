# Stable Branch Release (f4x)

Do this only after the Rawhide automatic Bodhi update is stable
(`koji latest-build f<NN> wasmedge` prints the new NVR). Repeat for each live
Fedora branch.

## Which branches are live

Ask Bodhi, not a hardcoded list and not the EOL wiki page:

```
bodhi releases list --rows 100
```

- Every `F<NN>` under `current` and `frozen` gets a build and a manual update
  (`frozen` is the branched, not-yet-released Fedora during its Final Freeze).
- `pending` holds Rawhide (already done) and the `F<NN>F` / `F<NN>C`
  Flatpak/Container variants, which are not dist-git branches — ignore them.
- Order newest → oldest, e.g. f45 → f44 → f43 in October 2026.

## The branch script

Each stable branch merges Rawhide, is `mock`-tested, then pushed, built, and given
a manual Bodhi update. Set `VERSION` and `BRANCH`, then:

```bash
#!/bin/bash
set -e
set -v

VERSION=<x.y.z>
BRANCH=45          # one of the live releases from `bodhi releases list`

echo "Release $VERSION at f$BRANCH..."
fedpkg switch-branch f$BRANCH
git merge rawhide
git log --oneline -3                    # HEAD = the rawhide "Release $VERSION" commit
fedpkg mockbuild                        # builds the SRPM and mock-tests it in the branch's chroot
git status
git branch
read -p "Ready to push? <Ctrl-C> to cancel"   # human gate
fedpkg push
fedpkg build
fedpkg verrel                           # must print wasmedge-$VERSION-1.fc$BRANCH
fedpkg update --type enhancement --notes "Release $VERSION"
```

## Notes

- **`git merge rawhide`** brings the version bump, the hygiene commits, and the
  lookaside metadata from the Rawhide branch — no need to re-edit the spec on
  Fedora stable branches. It has always been a fast-forward for wasmedge (older
  branches also pick up mass-rebuild commits, harmless under `%autorelease`).
  Confirm first: `git merge-base --is-ancestor origin/f$BRANCH origin/rawhide`.
- **`fedpkg mockbuild`** builds the SRPM *and* mock-tests it in the branch's chroot
  in one step (auto-selecting `fedora-$BRANCH-<arch>`). It compiles both cmake
  configurations, so expect roughly twice the `fedpkg local` time (2.5 minutes on an
  Apple Silicon host for 0.18.0). Fix any failure before pushing; it is far cheaper
  than a failed Koji build.
- **mock inside a container** pulls a bootstrap image with podman by default. If it
  dies before compiling anything, re-run `fedpkg mockbuild -- --no-bootstrap-image`.
- **mock arch** is the builder's native arch — a valid smoke test, since Koji builds
  all arches anyway.
- **`fedpkg update`** is required: stable branches do NOT auto-open a Bodhi update.
  The `--type enhancement --notes` form creates it without opening an editor and
  prints the `FEDORA-YYYY-...` ID. The first call logs in to Bodhi; later calls
  reuse the cached token. If it fails with `RemoteDisconnected`, check
  `bodhi updates query --builds wasmedge-$VERSION-1.fc$BRANCH` (nothing created →
  retry once); if the login hangs on a localhost callback, see the EPEL skill's
  gotchas (OIDC localhost callback).
- If `fedpkg update` keeps failing, create the update on the Bodhi web UI:
  https://bodhi.fedoraproject.org/updates/new

## What to expect afterwards

- `current` releases: the update sits in `testing` and reaches stable on autotime
  after 7 days with karma 0 (every wasmedge update since 2025 went this way), or
  earlier with +3 karma. `CI Status: ... 1 result missing` / `gating: waiting`
  right after submission is Fedora CI still running; it passes within the hour.
- `frozen` release: the update stays in `testing` until the Final Freeze lifts, then
  lands with no action from you.

## After all branches

Stable branches done. For RHEL/CentOS, continue with the
`wasmedge:epel-package-upgrade` skill (EPEL 8/9/10), from the same container.
