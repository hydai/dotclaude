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

EPEL 9 and 10 *support* `%autorelease`/`%autochangelog`, but support is not use:
the wasmedge **epel10** spec uses them, the wasmedge **epel9 and epel8** specs use a
literal `Release: 1%{?dist}` and a hand-maintained `%changelog` (state at 0.18.0).
Always check the branch's existing spec before deciding:

```
fedpkg switch-branch epel<N>
grep -nE 'autorelease|autochangelog|^Release:' wasmedge.spec
```

- `Release: %autorelease` + `%autochangelog`: nothing to do beyond a good commit
  message (`git commit -s -m 'Release <x.y.z>'`).
- Literal `Release:` + hand-maintained `%changelog`: insert the entry at the top of
  `%changelog`; `Release:` stays `1%{?dist}` for a new version. The weekday must
  match the date or rpmbuild warns `bogus date in %changelog`, so let `date` write it:

```
sed -i "/^%changelog$/a * $(date +'%a %b %d %Y') <MAINTAINER_NAME> <<MAINTAINER_EMAIL>> - <x.y.z>-1\n- Update to <x.y.z>\n" wasmedge.spec
sed -n '/^%changelog/,+4p' wasmedge.spec
git commit -s -m 'Update to <x.y.z>'
```

## Diverging branches: never `git merge rawhide`

Every EPEL branch has diverged from rawhide: a lagging version, platform-specific
LLVM (RHEL 10 → `llvm19` compat from EPEL, RHEL 9/8 → system `llvm-devel`,
rawhide → `llvm18`), `ExclusiveArch` without `riscv64`, `gcc-toolset-11` on
epel8, legacy `License:` strings, and a different changelog style. On 2026-10-06
`git merge-tree` reported conflicts in `wasmedge.spec`, `sources`, and
`.gitignore` for all five branches. Replay the bump by hand instead:

```
sed -i 's/^%global version <old>$/%global version <x.y.z>/' wasmedge.spec
git checkout rawhide -- sources .gitignore      # tarball already in the per-package lookaside
diff <(git show rawhide:wasmedge.spec) wasmedge.spec
```

Read the diff line by line:

- **KEEP** the platform lines: LLVM major in `BuildRequires` / `Requires`,
  `-DLLVM_DIR=.../llvmNN/...`, `ExclusiveArch`, `gcc-toolset-11-toolchain` and
  `source /opt/rh/gcc-toolset-11/enable`, `%undefine _annotated_build`, the legacy
  `License:` string, the literal `Release:` and `%changelog`.
- **PORT** everything else: new `%files` entries, build flags, and whatever the
  `spec-hygiene.md` commits changed on rawhide this release (`bundled()`
  versions, renamed CMake options). When rawhide's spec changed nothing but the
  version line (0.17.2, 0.18.0), the diff shows only platform lines and you are done.

Concrete trap from 0.17.0: it added 13 `%{_includedir}/%{name}/wasmedge_*.h`
headers that an older branch's `%files` lacked. A manual bump that skips this diff
fails `check-files` on 13 unpackaged headers. (`%files` order is irrelevant to RPM.)

`epel10.N` minor branches are the exception: they are byte-identical to `epel10`,
so `git merge epel10` is a fast-forward.

### Drop patches already upstream

A downstream patch may already be in the new tag. Confirm and drop it:

```
git merge-base --is-ancestor <commit> <newtag>   # exit 0 → already upstream, drop the patch
```

## Mock from the rawhide container (default), and why Stream 8 lies

The Fedora rawhide container's `mock-core-configs` ships the EPEL chroots, so no
Stream container is needed:

```
fedpkg mockbuild                                        # epel10: auto-selects epel-10-<arch>
fedpkg mockbuild --root centos-stream+epel-9-<arch>     # epel9: no epel-9-<arch>.cfg exists
fedpkg mockbuild --root centos-stream+epel-8-<arch>     # epel8: advisory only, see below
```

fedpkg derives the config name as `epel-<N>-<arch>` from the branch; only
`epel-10-<arch>.cfg` exists (an alias of `centos-stream+epel-10`), hence `--root`
for 9 and 8. `epel10.N` has no config at all; skip mock there (same spec as
`epel10`, built by Koji against the RHEL 10.N / snapshot repos).

How faithful each chroot is to Koji:

| Branch | mock chroot | Koji buildroot |
|:-------|:------------|:---------------|
| epel10 | CentOS Stream 10 build repo (kojihub.stream.centos.org) + EPEL 10 | `epel10.N-build`: the same CentOS Stream 10 repos |
| epel9 | CentOS Stream 9 build repo + EPEL 9 | RHEL 9: close (Stream is RHEL 9.next) |
| epel8 | CentOS Stream 8 **EOL snapshot via vault** (LLVM 17.0.6, gcc-toolset-11) | RHEL 8.10, which kept LLVM rebases (llvm-devel 21.1.8 in 2026) |

An epel8 mock failure that names an LLVM C API symbol (`LLVMSetTailCallKind` on
0.18.0) is the stale chroot, not the package: push and `fedpkg build`. Check what
Koji really has by reading the previous el8 build's buildroot:

```
koji rpminfo wasmedge-<prev>-1.el8.x86_64 | grep Buildroot
koji list-buildroot <buildroot-id> | grep -E '^(llvm|lld|gcc-toolset)'
```

If mock dies before compiling because podman cannot pull the bootstrap image, add
`-- --no-bootstrap-image` to the `fedpkg mockbuild` line.

## LLVM floor of the new release

The AOT compiler's minimum LLVM moves without a release note. Before an EPEL
build, read the guards in the new tag and compare them with each buildroot's LLVM:

```
git show "${TAG}:lib/llvm/llvm.h" | grep -n 'LLVM_VERSION_MAJOR'
```

0.18.0 uses `LLVMSetTailCallKind` (LLVM 18 C API) behind a shim guarded with
`#if LLVM_VERSION_MAJOR < 17` in `lib/llvm/llvm.h` and `lib/llvm/llvm.cpp`, so
LLVM ≤ 16 and ≥ 18 build but 17 does not — an upstream off-by-one (PR #4723
introduced it, #5189 and #5040 added call sites, #5101 moved the definitions).
EPEL buildroots in 2026 (el10 `llvm19`, el9 and el8 system LLVM 21) are above the
gap; only the Stream 8 chroot sits in it.

## Manual Bodhi update (and the web fallback)

EPEL never auto-opens a Bodhi update. After a successful Koji build:

```
fedpkg verrel                                                     # wasmedge-<x.y.z>-1.el<N>
fedpkg update --type enhancement --notes 'Update to <x.y.z>'     # no editor
```

This works for every EPEL branch, epel8 included, as long as the `bodhi-client`
doing it is ≥ 6.0.0 (the rawhide container's is). Only a Stream 8 container's own
client is too old; from there use the web UI:
https://bodhi.fedoraproject.org/updates/new — enter the build NVR
(`wasmedge-<x.y.z>-<rel>.el<N>`), set type=enhancement, add notes, Submit.

## Transient infrastructure errors: retry once

Seen on 2026-10-06 and 2026-07-06, each fixed by an immediate retry:

- `fedpkg update` → `Could not generate update request: ('Connection aborted.',
  RemoteDisconnected(...))`. Bodhi dropped the connection. Before retrying, make
  sure no update was created: `bodhi updates query --builds <nvr>` must return 0.
- `fedpkg push` → `remote: Unspecified ref refs/heads/epel9 is blocked` /
  `Denied push for ref`. The dist-git pre-receive hook (`pagure-dist-git`,
  `dist_git_auth.py`) asks Bodhi's `/releases/` whether the branch is active; when
  that request fails it returns "unknown" and the branch falls through to the
  unspecified-ref blacklist. Other branches pushed minutes before and after. Retry.
  It is only real if Bodhi no longer lists the release as current/pending/frozen
  (`bodhi releases list`) or the branch carries `dead.package`.

## Containerized Bodhi login: the OIDC localhost callback

`fedpkg push` is git over SSH and never needs a browser. The Bodhi CLI
(`fedpkg update`, `bodhi ...`) does: on first use it starts a one-shot HTTP server
on `127.0.0.1:<port>` **inside the container**, opens an authorization URL, and
waits. After you approve in your **host** browser, Fedora redirects it to
`http://localhost:<port>/?code=…` — the *host's* loopback, so the code never
reaches the container and the command hangs. In 2026 the login completed without
intervention twice ("Login successful!"); if it does hang, deliver the code to the
container's own loopback from a second shell:

```
docker exec <container> curl -s '<the full http://localhost:PORT/?code=… the browser landed on>'
```

- The **port changes every run** — copy the whole URL from the browser address bar.
- OIDC codes **expire in seconds** — stage the `docker exec` command before approving;
  if it expired, re-run `fedpkg update`, approve, and deliver the fresh code immediately.
- No `curl` in the image → `docker exec <container> python3 -c "import urllib.request;urllib.request.urlopen('<url>')"`.
- The token is cached afterwards; later `fedpkg update` calls do not prompt.

## Dependency-version walls (system fmt / spdlog too old)

Older EPEL ships system libraries (fmt, spdlog, …) older than upstream's pins:
RHEL 8 fmt 6.2.1 / spdlog 1.5, RHEL 9 fmt 8.1.1 / spdlog 1.10. Since 0.17.1 the
guards are upstream (`include/common/fmt.h`, `int128.h`, `spdlog.cpp`;
0.18.0 advertises fmt 6 to 12.2), so no downstream patch is needed and both
branches build patch-free. Keep the history for the next wall:

The build error names the missing header/symbol (e.g. a missing
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
  EPEL 8. Every hunk was conditional, which is why it could be upstreamed in 0.17.1.
