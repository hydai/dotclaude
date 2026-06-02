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

## Changelog: check the branch's spec first

EPEL 9/10 **do** support `%autorelease`/`%autochangelog` — treat them like a
Fedora stable branch: the commit message *is* the changelog, so do not
hand-write a `%changelog` entry. Only older **epel8** (or any branch whose spec
predates rpmautospec) needs a manual changelog. Always check the branch's
existing spec before deciding:

```
fedpkg switch-branch epel<N>
grep -nE 'autorelease|autochangelog|^Release:' wasmedge.spec
```

- If it shows `Release: %autorelease` + `%autochangelog`: nothing to do beyond a
  good commit message (`git commit -s -m 'Update to <x.y.z>'`).
- If it shows a literal `Release:` and a hand-maintained `%changelog`: use the
  epel8/legacy path below.

### epel8 / legacy: hand-written changelog

```
fedpkg switch-branch epel<N>
git merge rawhide          # but see "Diverging branches" below — rarely clean
vim wasmedge.spec          # add a dated %changelog entry by hand
git add -p
git commit -s -m 'Update changelog'
# then resume the loop: fedpkg local → (optional mock) → fedpkg push → fedpkg build
```

If a merge carried the version bump and lookaside metadata, do not re-edit
`Version:` or re-run `fedpkg upload`.

## Diverging branches: `git merge rawhide` is rarely clean

EPEL branches frequently **diverge** from rawhide: a lagging version,
platform-specific LLVM (e.g. RHEL 10 → llvm19 vs rawhide's llvm18), different
downstream patches, and a different `ExclusiveArch` (e.g. no `riscv64` on RHEL
10). Do **not** assume `git merge rawhide` brings only the version bump. Pick one:

- **(a) Manual bump + reconcile.** Edit `%global version` / `%global
  capi_version` by hand, review patches (see below), then reconcile the spec body
  against rawhide (next step).
- **(b) Careful merge.** `git merge rawhide` and resolve every conflict so the
  EPEL platform lines (LLVM version, `LLVM_DIR`, `ExclusiveArch`) survive while
  taking rawhide's version and body changes. One slip on an LLVM line → unbuildable.

### Required: reconcile the spec body vs rawhide

After the version bump, diff the whole body against rawhide so nothing silently
drifts:

```
diff <(git show origin/rawhide:wasmedge.spec) wasmedge.spec
```

- **KEEP** the platform-specific lines: the LLVM version in `BuildRequires` /
  `Requires`, `-DLLVM_DIR=.../llvmNN/...`, and `ExclusiveArch`.
- **PORT** everything else: new `%files` entries, build flags, etc.

Concrete trap from 0.17.0: it added 13
`%{_includedir}/%{name}/wasmedge_*.h` headers that an older branch's `%files`
lacked. A manual bump that skips this diff fails `fedpkg local` `check-files` on
13 unpackaged headers. (`%files` order is irrelevant to RPM.)

### Drop patches already upstream

A downstream patch may already be in the new tag. Confirm and drop it:

```
git merge-base --is-ancestor <commit> <newtag>   # exit 0 → already upstream, drop the patch
```

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

## Dependency-version walls (system fmt / spdlog too old)

Older EPEL may ship system libraries (fmt, spdlog, …) too old for the new
upstream. The build error names the missing header/symbol (e.g. a missing
`<spdlog/sinks/callback_sink.h>` or `fmt::detail::uint128_fallback`). Two options:

- **(a)** Keep EPEL at the last buildable upstream version.
- **(b)** Write a **conditional** downstream compat patch guarded by
  `#if <LIB>_VERSION …` / `__has_include(...)`. It compiles away on Fedora's newer
  libs, so it is also upstreamable.

Do **not** rely on the project's `FetchContent` fallback to pull newer fmt/spdlog:
Koji buildroots have **no network**, so that path only works locally.

Rule of thumb: a small guard patch only works when the system lib is *one* major
behind. When system fmt/spdlog is **several** majors behind, it is not enough —
stay on the last buildable version. Two concrete outcomes from 0.17.0:

- **RHEL 9 — buildable with a small guard patch (shipped).** fmt 8.1.1 /
  spdlog 1.10. A ~2-file guard was enough: `#if FMT_VERSION >= 90000` around the
  int128 fmt block, plus an `__has_include(<spdlog/sinks/callback_sink.h>)`
  fallback that provides a small `callback_sink`. Built clean on Koji.
- **RHEL 8 — too old; guard patch does NOT scale (stays on 0.14.0).** fmt 6.2.1 /
  spdlog 1.5. A raw 0.17.0 build produced **375 fmt-6 errors** — not just the
  int128 detail block but the int128 *formatter* template and the enum/type
  formatters (`fmt::formatter<ValType|WasmPhase|ErrCode|…>`), whose fmt-6 API is
  incompatible. This matches the scale of the branch's existing
  `wasmedge-revert-fmt.patch` (~1664 lines). epel8 stays on the last buildable
  version.
