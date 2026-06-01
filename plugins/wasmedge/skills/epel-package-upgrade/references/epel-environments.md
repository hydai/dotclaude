# EPEL Container Environments (8 / 9 / 10)

Each EPEL version builds on a different CentOS Stream image with different repo
setup. Run these before the Fedora maintainer setup (git identity, SSH key,
`fkinit`, `fedpkg co wasmedge`). All containers need `--privileged`.

> `fedora-review` and `fedora-packager-kerberos` are not available on CentOS Stream.
> That is expected — install `fedora-packager git vim` and continue.

## EPEL 8 — `quay.io/centos/centos:stream8`

CentOS 8 has reached end of life, so the default mirrors are gone — repoint the
repos at `vault.centos.org` first. `glibc-langpack-en` silences a locale warning.

```
docker pull quay.io/centos/centos:stream8
docker run -it --rm --privileged --name wasmedge-epel8 -h wasmedge-epel8 quay.io/centos/centos:stream8

sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
dnf install -y 'dnf-command(config-manager)'
dnf config-manager --set-enabled powertools
dnf install -y epel-release epel-next-release glibc-langpack-en
```

## EPEL 9 — `quay.io/centos/centos:stream9`

`crb` (CodeReady Builder) replaces `powertools` on Stream 9.

```
docker pull quay.io/centos/centos:stream9
docker run -it --rm --privileged --name wasmedge-epel9 -h wasmedge-epel9 quay.io/centos/centos:stream9

dnf install -y 'dnf-command(config-manager)'
dnf config-manager --set-enabled crb
dnf install -y epel-release epel-next-release
```

## EPEL 10 — `quay.io/centos/centos:stream10`

EPEL 10 is installed from an RPM URL (no `epel-next-release`).

```
docker pull quay.io/centos/centos:stream10
docker run -it --rm --privileged --name wasmedge-epel10 -h wasmedge-epel10 quay.io/centos/centos:stream10

dnf install -y 'dnf-command(config-manager)'
dnf config-manager --set-enabled crb
dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
```

## Next

Install the packager tools (`dnf install -y fedora-packager git vim`), do the
maintainer setup, then continue with `epel-gotchas.md` and the shared build loop.
