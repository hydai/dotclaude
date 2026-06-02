# Stable Branch Release (f4x)

Do this only after the Rawhide Bodhi update is submitted. Repeat for each live
Fedora branch, and skip branches that have reached end of life.

> Check which branches are still supported before you start:
> https://docs.fedoraproject.org/en-US/releases/eol/

## The branch script

Each stable branch merges Rawhide, is `mock`-tested, then pushed, built, and given
a manual Bodhi update. Set `VERSION` and `BRANCH`, then:

```bash
#!/bin/bash
set -e
set -v

VERSION=<x.y.z>
BRANCH=41          # a live, non-EOL Fedora release (e.g. 41, 40, …)

echo "Release $VERSION at f$BRANCH..."
fedpkg switch-branch f$BRANCH
git merge rawhide
fedpkg mockbuild      # builds the SRPM and mock-tests it in the branch's chroot
git status
git branch
read -p "Ready to push? <Ctrl-C> to cancel"   # human gate
fedpkg push
fedpkg build
fedpkg update      # type=enhancement (human step)
```

## Notes

- **`git merge rawhide`** brings the version bump and lookaside metadata from the
  Rawhide branch — no need to re-edit the spec on Fedora stable branches.
- **`fedpkg mockbuild`** builds the SRPM *and* mock-tests it in the branch's chroot
  in one step (auto-selecting the branch's mock config) — so a clean checkout needs no
  separate SRPM build. Fix any failure before pushing; it is far cheaper than a failed
  Koji build. (To keep raw `mock`, run `fedpkg srpm` first to produce the
  `wasmedge-$VERSION-*.fc$BRANCH.src.rpm` it expects.)
- **mock arch** is the builder's native arch (not a hardcoded `-x86_64.cfg`) when using
  `fedpkg mockbuild` — a valid smoke test, since Koji builds all arches anyway.
- **`fedpkg update`** is required: stable branches do NOT auto-open a Bodhi update.
  Choose `type=enhancement` for a version upgrade.
- If `fedpkg update` fails, create the update directly on the Bodhi web UI:
  https://bodhi.fedoraproject.org

## After all branches

Stable branches done. For RHEL/CentOS, continue with the
`wasmedge:epel-package-upgrade` skill (EPEL 8/9/10).
