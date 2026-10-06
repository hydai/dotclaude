# Spec Hygiene Checklist — run on every version bump

The version line is never the only thing to check. Run these against the new
upstream tag from a WasmEdge checkout (`git fetch --tags`), with
`TAG=<x.y.z>` and `PREV=<previous x.y.z>` exported, and the dist-git branch's
`wasmedge.spec` at hand. Do it once on `rawhide`; stable branches inherit the
result through `git merge rawhide`, and the EPEL branches replay it by hand (see
`wasmedge:epel-package-upgrade`).

**One concern per commit.** The `Release <x.y.z>` commit carries only the version
line, `sources`, and `.gitignore`. Each drifted item below gets its own commit
(`Fix bundled(blake3) version`, `Use WASMEDGE_USE_LLVM for the rt build`, ...), so
`%autochangelog` and `git blame` stay readable.

Write `${TAG}:path` with braces: in zsh a bare `$TAG:t` is a path modifier.

| # | Check | Command | When it drifts |
|:--|:------|:--------|:---------------|
| 1 | C API version / soname | `git show "${TAG}:CMakeLists.txt" \| grep -E 'WASMEDGE_CAPI_(VERSION\|SOVERSION)'` vs `%global capi_version` / `%global capi_soname` | Bump both macros. Symptom of a miss: `%install` fails with `mv: cannot stat '.../libwasmedge.so.<old>'` |
| 2 | Installed headers (`%files devel`) | `git diff --name-status --diff-filter=ADR "$PREV" "$TAG" -- include/api/wasmedge/` and `git diff "$PREV" "$TAG" -- lib/api/CMakeLists.txt \| grep -E '^[+-].*\.(h\|inc)$'` (the `WASMEDGE_CAPI_HEADERS` install list) | Add or drop `%files devel` lines. The local build is the authority: `Installed (but unpackaged) file(s) found` → add; `File not found` → remove. 0.17.0 added 13 headers this way |
| 3 | Bundled blake3 version | `git show "${TAG}:thirdparty/blake3/blake3.h" \| grep BLAKE3_VERSION_STRING` vs **both** `Provides: bundled(blake3) =` lines (main and `rt` subpackage) | Set both to the header value. 0.18.0 ships 1.5.0; the spec still said 1.2.0 / 1.3.3 |
| 4 | Other vendored code | `git ls-tree --name-only "$TAG" thirdparty/` vs the `bundled()` Provides (`wasi-cpp-header` is `thirdparty/wasi/api.hpp`) | Add `Provides: bundled(<name>) = <ver>` for anything new the build links |
| 5 | CMake options the spec passes | `for o in $(grep -oE 'WASMEDGE_[A-Z_]+' wasmedge.spec \| sort -u); do git show "${TAG}:CMakeLists.txt" \| grep -qE "^option\($o " \|\| echo "not an option: $o"; done` | Switch to the current name. `WASMEDGE_BUILD_AOT_RUNTIME=OFF` is a rename shim for `WASMEDGE_USE_LLVM=OFF` (still honoured in 0.18.0, with a warning) |
| 6 | LLVM range vs the pin | `git show "${TAG}:lib/llvm/llvm.h" \| grep -n LLVM_VERSION_MAJOR` and `git show "${TAG}:cmake/Helper.cmake" \| grep -n LLVM_VERSION_MAJOR`; compare with `llvmNN-devel` in `BuildRequires` and with each EPEL buildroot's LLVM | Bump the pinned major in `BuildRequires`, `Requires`, and both `-DLLVM_DIR=` lines. A guard whose threshold is off by one is an upstream bug: 0.18.0 guards the `LLVMTailCallKind` shim with `< 17` although the C API needs 18, so LLVM 17 cannot build it |
| 7 | `%license` / `%doc` files | `git ls-tree --name-only "$TAG" LICENSE LICENSE.spdx SECURITY.md Changelog.md README.md` | Drop or rename the `%files` entries |
| 8 | Downstream patches | `git merge-base --is-ancestor <patch-commit> "$TAG" && echo DROP` | Remove `PatchN:` and `git rm` the file (see `rawhide-release.md`, Patch review) |
| 9 | Upstream changelog scan | `git show "${TAG}:Changelog.md" \| awk "/^### $TAG/{f=1} /^### $PREV/{f=0} f" \| grep -niE 'cmake\|llvm\|remov\|deprecat\|install\|fmt\|spdlog\|plug-in'` | Anything that touches the build, the CLI, or the installed files becomes a spec change or a note in the update |
| 10 | Lookaside metadata | `cat sources` has exactly one line (`fedpkg new-sources`); `.gitignore` may keep old names | Nothing to fix; trim `.gitignore` only if you want |
| 11 | License tag | Fedora branches use SPDX (`Apache-2.0 AND CC0-1.0`); EPEL 9/8 keep `ASL 2.0 and CC0` | Leave EPEL alone; the rpm there predates SPDX validation |

Items 1, 2, 6, 7, and 8 break the build when wrong. Items 3, 4, 5, and 11 are
metadata correctness and ride along as separate commits in the same release.
Item 9 is what tells you about a removed CLI flag or plug-in before a user does.

Known drift at 0.18.0 (carry these as follow-up commits on the next release if not
done yet): item 3 (`bundled(blake3)` 1.2.0 / 1.3.3 → 1.5.0) and item 5
(`WASMEDGE_BUILD_AOT_RUNTIME=OFF` → `WASMEDGE_USE_LLVM=OFF` in the rt build).
