# Environment Setup — Fedora Packaging Container

One-time setup for a build container and maintainer credentials. Re-run the
container steps whenever you start a fresh container; fill in the placeholders
once per maintainer.

## Fill these in once

| Placeholder | Meaning | Example |
|:------------|:--------|:--------|
| `<FAS_USERNAME>` | Your Fedora Account System (FAS) username, used by `fkinit` | `jdoe` |
| `<MAINTAINER_NAME>` | `git` author name | `Jane Doe` |
| `<MAINTAINER_EMAIL>` | `git` author email | `jane@example.org` |

## Start a privileged Rawhide container

`--privileged` is required — `mock` and `fedpkg local` build inside a chroot.

```
docker pull fedora:rawhide
docker run -it --rm --privileged --name wasmedge-rawhide -h wasmedge-rawhide fedora:rawhide
```

## Install the packager toolchain

```
cd ~
dnf install -y --skip-broken fedora-packager fedora-review fedora-packager-kerberos git vim
```

## Fix the Kerberos ccache

`fkinit` misbehaves unless the default ccache name is disabled:

```
sed -i 's/default_ccache_name/#default_ccache_name/' /etc/krb5.conf
```

## Add your SSH key (human step)

`fedpkg push` talks to `src.fedoraproject.org` over SSH. The key MUST be `chmod 400`.

```
mkdir -p ~/.ssh
vim ~/.ssh/id_rsa          # paste your private key
chmod 400 ~/.ssh/id_rsa
```

## Set git identity and log in to FAS

```
git config --global user.email <MAINTAINER_EMAIL>
git config --global user.name <MAINTAINER_NAME>
fkinit -u <FAS_USERNAME>   # Kerberos login, prompts for your FAS password (human step)
```

## Check out the package

```
fedpkg co wasmedge
cd wasmedge
```

You are now on the `rawhide` branch by default. Continue with `rawhide-release.md`.

> EPEL note: EPEL uses CentOS Stream base images and a different setup — see the
> `wasmedge:epel-package-upgrade` skill, not this file. On those images
> `fedora-review` and `fedora-packager-kerberos` are unavailable, and that is expected.
