# EPEL Container Environments (8 / 9 / 10) — fallback only

**Default is not this file.** The EPEL stage runs from the Fedora rawhide container
with `fedpkg mockbuild` against the `epel-10-<arch>` / `centos-stream+epel-9-<arch>`
/ `centos-stream+epel-8-<arch>` chroots (see `SKILL.md` Step 2). Use the
per-version containers below only when mock is unavailable, or to debug inside a
Stream userland. Everything that follows keeps the traps that bit on the way.

Each EPEL version builds on a different CentOS Stream image with different repo
setup. Run these before the Fedora maintainer setup (git identity, SSH key,
`fkinit`, `fedpkg co wasmedge`). All containers need `--privileged`; none should
use `--rm` (a release spans hours and several terminals).

> `fedora-review` may not be available on CentOS Stream, but
> `fedora-packager-kerberos` (which provides `fkinit`) **is** available on EPEL
> 9/10 — install `fedora-packager git vim` and continue.

### Authentication — use `fkinit`, not plain `kinit`

Authenticate with `fkinit -u <FAS_USERNAME>` (the `-u` is required — bare `fkinit`
builds an empty principal `@FEDORAPROJECT.ORG`). Do **not** use plain `kinit`.

```
fkinit -u <FAS_USERNAME>
```

- If the account has 2FA, the prompt asks for "password and OTP concatenated" —
  enter `<password><OTP>` with **no space**.
- Plain `kinit <FAS>@FEDORAPROJECT.ORG` fails with `Pre-authentication failed:
  Invalid argument`. This is because Fedora's KDC requires **FAST armor**, which
  `fkinit` sets up via anonymous PKINIT — it is **NOT** a ccache (KEYRING/KCM) or
  crypto-policy problem (those are red herrings).
- Diagnose with `KRB5_TRACE=/dev/stdout kinit ...` if needed (it shows the KDC
  offering only PKINIT preauth, no password preauth).

## EPEL 8 — `quay.io/centos/centos:stream8`

CentOS 8 has reached end of life, so the default mirrors are gone — repoint the
repos at `vault.centos.org` first. `glibc-langpack-en` silences a locale warning.

**This userland is frozen at the May 2024 snapshot** (LLVM 17.0.6, gcc-toolset-11),
while Koji builds `epel8` against RHEL 8.10, which kept receiving LLVM rebases
(21.1.8 in 2026). A `fedpkg local` here that fails on an LLVM API is a false
negative — the same caveat as the Stream 8 mock chroot in `epel-gotchas.md`.

```
docker pull quay.io/centos/centos:stream8
docker run -it --privileged --name wasmedge-epel8 -h wasmedge-epel8 quay.io/centos/centos:stream8

sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
dnf install -y 'dnf-command(config-manager)'
dnf config-manager --set-enabled powertools
dnf repolist | grep -iE 'crb|powertools|epel'   # VERIFY powertools actually enabled
dnf install -y epel-release epel-next-release glibc-langpack-en
```

`ninja-build` (and other build tools) come from **powertools**, so it is
mandatory. If `--set-enabled powertools` did not persist, pass it per-transaction
on `dnf builddep` instead — this is the reliable form:

```
dnf builddep -y --enablerepo=powertools wasmedge.spec
```

(epel8 uses a manual `%changelog`, so `rpmautospec` is **not** needed here. Its
`bodhi-client` is `< 6.0.0`, so `fedpkg update` must run elsewhere or go through
the Bodhi web UI.)

## EPEL 9 — `quay.io/centos/centos:stream9`

`crb` (CodeReady Builder) replaces `powertools` on Stream 9.

```
docker pull quay.io/centos/centos:stream9
docker run -it --privileged --name wasmedge-epel9 -h wasmedge-epel9 quay.io/centos/centos:stream9

dnf install -y 'dnf-command(config-manager)'
dnf config-manager --set-enabled crb
dnf repolist | grep -iE 'crb|powertools|epel'   # VERIFY crb actually enabled
# epel-release MUST be installed BEFORE any EPEL-only package. dnf resolves every
# argument against the currently-enabled repos up front, so bundling the EPEL-only
# rpmautospec here fails ("No match for argument: rpmautospec") and aborts the whole
# transaction -- epel-release included. Enable EPEL first, then install from it.
dnf install -y epel-release epel-next-release
dnf install -y rpmautospec
```

`ninja-build` (and other build tools) come from **crb**, so it is mandatory. If
`--set-enabled crb` did not persist, pass it per-transaction on `dnf builddep`
instead — this is the reliable form:

```
dnf builddep -y --enablerepo=crb wasmedge.spec
```

(`rpmautospec` lets `fedpkg local`/mock expand `%autorelease` / `%autochangelog`
where a spec uses them; the wasmedge epel9 spec does not, epel10 does.)

## EPEL 10 — `quay.io/centos/centos:stream10`

EPEL 10 is installed from an RPM URL (no `epel-next-release`).

```
docker pull quay.io/centos/centos:stream10
docker run -it --privileged --name wasmedge-epel10 -h wasmedge-epel10 quay.io/centos/centos:stream10

dnf install -y 'dnf-command(config-manager)'
dnf config-manager --set-enabled crb
dnf repolist | grep -iE 'crb|powertools|epel'   # VERIFY crb actually enabled
dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
dnf install -y rpmautospec
```

On CentOS Stream 10 / dnf5 the `--set-enabled crb` sometimes does **not** persist
(`dnf repolist` will not list it). `ninja-build` (and other build tools) come from
**crb**, so it is mandatory — pass `--enablerepo=crb` per-transaction on
`dnf builddep` as the reliable form:

```
dnf builddep -y --enablerepo=crb wasmedge.spec
```

(`rpmautospec` is installed so `fedpkg local`/mock can expand `%autorelease` /
`%autochangelog`, which the EPEL 10 spec uses.)

## Next

Install the packager tools (`dnf install -y fedora-packager git vim`), do the
maintainer setup (authenticate with `fkinit -u <FAS_USERNAME>` — see
*Authentication* above), then continue with `epel-gotchas.md` and the shared
build loop.
