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
Do **not** add `--rm`: a release (Rawhide, three stable branches, five EPEL
branches) takes hours across several terminals, and `--rm` throws away the
checkout and the installed build deps the moment the shell exits.

```
docker pull fedora:rawhide
docker run -it --privileged --name wasmedge-rawhide -h wasmedge-rawhide fedora:rawhide
```

- Second terminal: `docker exec -it wasmedge-rawhide bash`
- Shell exited: `docker start -ai wasmedge-rawhide`
- Whole release done (EPEL included): `docker rm -f wasmedge-rawhide`

## Install the packager toolchain

Rawhide is dnf5: `dnf builddep` lives in `dnf5-plugins` (`dnf-plugins-core` is the
dnf4 name), and dnf5's `--skip-broken` does **not** skip an unknown package name —
`--skip-unavailable` does. `vim-enhanced` is the package that provides `vim`.

```
cd ~
dnf install -y --skip-unavailable fedora-packager fedora-review fedora-packager-kerberos dnf5-plugins git vim-enhanced
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

The ticket expires the next day (`klist` shows the exact expiry; one observed
ticket ran 28 hours and was renewable for 7 days). A release that continues the
next day needs `fkinit -u <FAS_USERNAME>` again before any `fedpkg new-sources`,
`fedpkg push`, or `fedpkg build`.

## Check out the package

```
fedpkg co wasmedge
cd wasmedge
```

You are now on the `rawhide` branch by default. Continue with `rawhide-release.md`.

> EPEL note: the EPEL stage runs from this same container with `fedpkg mockbuild`
> against the `centos-stream+epel-N` mock configs it already ships — see the
> `wasmedge:epel-package-upgrade` skill. The per-version CentOS Stream containers in
> that skill's `epel-environments.md` are the fallback, not the default.
