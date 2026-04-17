# Recipe-only Debian packaging for Linux 7.0

`linux_recipe/` is a packaging-only Git repository for a Milk-V Duo 256M
kernel package built from upstream Linux `7.0`, published under the
`cv181x-sodoport` flavour name.

The repository intentionally does not contain unpacked upstream sources.

**Branch model**

- `master`: recipe-only branch. CI builds artifacts, regenerates the published
  `gbp` branches, and pushes them back to GitHub.
- `latest-recipe`: recipe-only branch. CI builds and publishes `.deb` files to
  `deb-s3` without updating the published `gbp` branches.
- `latest`: generated build branch with unpacked upstream sources plus
  `debian/`.
- `upstream/latest`: imported upstream source branch for `gbp`.
- `pristine-tar`: `pristine-tar` metadata.

The generated `latest` branch is reconstructed by CI with `uscan` and
`gbp import-orig`, or by reapplying `debian/` onto the existing imported
upstream branch when the upstream version has not changed.

The initial board configuration and DTS seed are derived from the local
`/root/uboot/sg2002` Alpine package draft, but the Debian recipe keeps only the
kernel-facing pieces:

- `debian/config/cv181x-sodoport.config` is a temporary generated seed
  translated from the Alpine `cv180x.riscv64.config`, with the Alpine
  localversion replaced by the `cv181x-sodoport` flavour suffix.
- `debian/patches/0001-riscv-dts-sophgo-add-sg2002-milkv-duo256m.patch`
  carries the board DTS addition for upstream `7.0`.
- `debian/config/base/` and `debian/config/fragments/` now hold the
  long-term config inputs. The intended Debian workflow is to replace the
  placeholder `debian-common.fragment` with one generated from the intersection
  of Debian kernel configs across architectures, then regenerate the full board
  config with `debian/scripts/update-config.sh`.

**Local build flow**

```sh
cd /root/uboot/linux_recipe
git fetch origin latest upstream/latest pristine-tar --tags || true
git worktree add ../linux-build origin/master
cd ../linux-build
version=$(dpkg-parsechangelog --show-field Version | sed 's/-[^-]*$//')
uscan --check-dirname-level 0 --download-current-version --rename --destdir ..
gbp import-orig --no-interactive --debian-branch=latest \
  --upstream-branch=upstream/latest --upstream-version "$version" \
  ../linux-cv181x-sodoport_${version}.orig.tar.xz
dpkg-buildpackage -us -uc -b -a riscv64
```

Once the upstream tree is imported, create or refresh patches with:

```sh
dpkg-source --commit
```

To regenerate the checked-in board config from the fragment stack after you
derive a real Debian `debian-common.fragment`, run:

```sh
debian/scripts/update-config.sh /path/to/linux
```

The Debian outputs are split as follows:

- `linux-image-7.0.0-cv181x-sodoport`: the versioned kernel package, which
  installs `/boot/vmlinuz-7.0.0-cv181x-sodoport`,
  `/boot/System.map-7.0.0-cv181x-sodoport`,
  `/boot/config-7.0.0-cv181x-sodoport`, board DTBs under
  `/usr/lib/linux-image-7.0.0-cv181x-sodoport/`, and modules under
  `/lib/modules/7.0.0-cv181x-sodoport/`.
- `linux-image-cv181x-sodoport`: an unversioned meta-package that depends on
  the current versioned kernel.
- `linux-source-cv181x-sodoport`: a compressed, patched source tree under
  `/usr/src/`.

The kernel package itself does not choose a board DTB. When `u-boot-menu` is
installed by a board support package, Debian kernel hooks still regenerate
`/boot/extlinux/extlinux.conf`; the board package is responsible for setting
`fdtfile` in U-Boot and for any `u-boot-menu` policy such as DTB syncing or
overlay paths.

**CI inputs**

- `DEB_S3_BUCKET` repo variable: required for `deb-s3` publishing.
- `DEB_S3_CODENAME`, `DEB_S3_COMPONENT`, `DEB_S3_REGION`, `DEB_S3_ENDPOINT`,
  `DEB_S3_FORCE_PATH_STYLE`, `DEB_S3_PREFIX`, `DEB_S3_ORIGIN`, `DEB_S3_SUITE`,
  `DEB_S3_CLEAN`, `DEB_S3_PRESERVE_VERSIONS`, `DEB_S3_LOCK`,
  `DEB_S3_FAIL_IF_EXISTS`, `DEB_S3_USE_SESSION_TOKEN`, `DEB_S3_VISIBILITY`,
  `DEB_S3_SIGN_KEY` repo variables: optional publish controls. Set
  `DEB_S3_SIGN_KEY` to a full fingerprint or key ID to make `deb-s3` sign both
  `InRelease` and `Release.gpg`.
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` secrets:
  used by `deb-s3`.
- `DEB_REPO_SIGNING_PRIVATE_KEY` secret: optional ASCII-armored or base64
  encoded private OpenPGP key for repository signing.
- `DEB_REPO_SIGNING_PASSPHRASE` secret: optional passphrase for the private
  key. CI passes it to `gpg` through a temporary file with loopback pinentry.

To create a dedicated repository signing key locally:

```sh
gpg --quick-gen-key 'Sodo Repo Signing <repo@example.com>' ed25519 sign 2y
key_id=$(gpg --list-secret-keys --with-colons 'Sodo Repo Signing <repo@example.com>' | awk -F: '/^fpr:/ { print $10; exit }')
gpg --armor --export-secret-keys "$key_id" > repository-signing-private.asc
gpg --armor --export "$key_id" > repository-signing-key.asc
gpg --export "$key_id" > repository-signing-key.gpg
```

Then configure GitHub as follows:

- repo variable `DEB_S3_SIGN_KEY`: set it to the fingerprint in `key_id`, or
  leave it unset and let CI use the first imported secret key.
- repo secret `DEB_REPO_SIGNING_PRIVATE_KEY`: paste the contents of
  `repository-signing-private.asc`.
- repo secret `DEB_REPO_SIGNING_PASSPHRASE`: set it only if the private key is
  passphrase protected.

When signing is enabled, the `latest-recipe` workflow also exports
`repository-signing-key.asc` and `repository-signing-key.gpg` into the uploaded
build artifact so clients can install the public key.

`ci/run-sbuild.sh` refreshes the Debian archive keyring from the official
Debian package pool when bootstrapping a Debian mirror.
