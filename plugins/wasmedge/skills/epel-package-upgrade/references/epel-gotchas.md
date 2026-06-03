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

If `fedpkg update` fails, create the update on the Bodhi web UI instead:
https://bodhi.fedoraproject.org/updates/new — enter the build NVR
(`wasmedge-<x.y.z>-<rel>.el<N>`), set type=enhancement, add notes, Submit.

On **EPEL 8 the web UI is mandatory, not a fallback**: Stream 8's `bodhi-client` is
`< 6.0.0`, so `fedpkg update` aborts with `bodhi-client < 6.0.0 is not supported`,
and el8 cannot provide 6.0+ (it needs Python 3.8+). The update is only an API call
referencing the Koji build, so the web UI works fine — and it also sidesteps the
OIDC callback issue below.

## Containerized `fedpkg push` / `update`: the OIDC localhost callback

`fedpkg push` (and CLI Bodhi auth) use OIDC: fedpkg starts a one-shot HTTP server on
`127.0.0.1:<port>` **inside the container**, opens an authorization URL, and waits.
After you approve in your **host** browser, Fedora redirects it to
`http://localhost:<port>/?code=…` — but that is the *host's* loopback, and the port
is not published into the container, so the code never reaches fedpkg and the command
hangs.

Deliver the code to the container's own loopback instead, from a second shell:

```
docker exec <container> curl -s '<the full http://localhost:PORT/?code=… the browser landed on>'
```

- The **port changes every run** — copy the whole URL from the browser address bar.
- OIDC codes **expire in seconds** — stage the `docker exec` command before approving;
  if it expired, re-run `fedpkg push`, approve, and deliver the fresh code immediately.
- No `curl` in the image → `docker exec <container> python3 -c "import urllib.request;urllib.request.urlopen('<url>')"`.

Root cause: OAuth's loopback redirect assumes the browser and the listener share one
network namespace; a host browser with an in-container listener breaks that. (You could
also run the container with `-p <port>:<port>`, but the port is unpredictable.)

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

Rule of thumb: the error *count* is misleading. What decides buildability is
whether the breakage is **mechanical** — const-ness, formatter context type, a
handful of missing `detail` symbols, all addressable with `#if`/`__has_include`
guards and template-ization — or **semantic** (real API/behavior changes a guard
cannot bridge). Triage the *kinds* of error, not the number; mechanical breakage
is rescuable even **several** majors behind. Two outcomes from 0.17.0, both shipped:

- **RHEL 9 — buildable with a small guard patch (shipped).** fmt 8.1.1 /
  spdlog 1.10. A ~2-file guard was enough: `#if FMT_VERSION >= 90000` around the
  int128 fmt block, plus an `__has_include(<spdlog/sinks/callback_sink.h>)`
  fallback that provides a small `callback_sink`. Built clean on Koji.
- **RHEL 8 — fmt 6.2.1 / spdlog 1.5, extended guard (shipped 0.17.0).** A raw build
  produced **375 fmt-6 errors**, but they were a few mechanical patterns repeated
  across many formatters, not 375 distinct problems. Three guards on top of the
  RHEL 9 patch took it 375 → 0:
  1. `WASMEDGE_FMT_CONST` — `const` on `FMT_VERSION >= 80000`, empty otherwise
     (fmt 8 made `formatter::format()` const-qualified).
  2. **Template the formatter context** — swap the hard-coded
     `fmt::format_context::iterator format(…, fmt::format_context &Ctx)` for
     `template <typename FmtCtx> auto format(…, FmtCtx &Ctx) … -> decltype(Ctx.out())`
     (fmt 6's `format_context` uses a different output-iterator type).
  3. An fmt-6 `#else` `uint128` formatter (the fmt 9+ `dynamic_format_specs` path is
     gone), plus `__has_include(<spdlog/fmt/ranges.h>)` → `<fmt/ranges.h>` (spdlog
     1.5 lacks the bundled header).

  ~127 lines over 13 files, **replacing** the branch's old 1664-line
  `wasmedge-revert-fmt.patch`. Built on Koji (x86_64 + aarch64) and shipped to
  EPEL 8. Every hunk is conditional, so it is also **upstreamable** — one PR makes
  WasmEdge build fmt 6→11, after which EPEL 8/9 need no downstream patch.
