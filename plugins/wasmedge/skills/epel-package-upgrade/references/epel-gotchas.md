# EPEL Gotchas

The EPEL-specific deltas layered on top of the Fedora release loop.

## Request a new EPEL branch (one-time)

If the `epel<N>` branch does not exist for the package yet, request it through
Pagure. You need a Pagure API token (`<PAGURE_TOKEN>`).

```
fedpkg set-pagure-token <PAGURE_TOKEN>
fedpkg request-branch --repo wasmedge epel8     # or epel9 / epel10
```

Wait for the branch to be created before checking it out.

## Manual changelog (no %autorelease)

EPEL does not support `%autorelease`/`%autochangelog`. After merging Rawhide you
must add the changelog entry by hand, then continue the build loop:

```
fedpkg switch-branch epel<N>
git merge rawhide
vim wasmedge.spec          # add a dated %changelog entry by hand
git add -p
git commit -s -m 'Update changelog'
# then resume the loop: fedpkg local → (optional mock) → fedpkg push → fedpkg build
```

The merge already carries the version bump and lookaside metadata, so do not
re-edit `Version:` or re-run `fedpkg upload`.

## Optional: mock test before pushing

To verify the build in a clean chroot (as the stable-branch flow does), `mock`-test
the SRPM before `fedpkg push`. Find the matching config name first — it varies by
`mock` version:

```
ls /etc/mock/ | grep -i epel
mock -r <epel-config-from-the-list> wasmedge-<x.y.z>-*.el<N>.src.rpm
```

Otherwise `fedpkg local` already builds the package for the branch target.

## Manual Bodhi update (and the web fallback)

EPEL never auto-opens a Bodhi update. After a successful Koji build:

```
fedpkg update      # type=enhancement (human step)
```

If `fedpkg update` fails, create the update directly on the Bodhi web UI:
https://bodhi.fedoraproject.org
