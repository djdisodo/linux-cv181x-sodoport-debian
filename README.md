# Recipe-only Debian packaging for Linux 7.0

`linux_recipe/` is a packaging-only Git repository for a Milk-V Duo 256M
kernel package built from upstream Linux `7.0`, published under the
`cv181x-sodoport` flavour name.

The repository intentionally does not contain unpacked upstream sources. The
expected workflow is:

1. Keep the recipe on `master`.
2. Import upstream tarballs into a generated `latest` branch with
   `gbp import-orig`.
3. Maintain board-specific DTS patches with `dpkg-source --commit`.

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

Suggested local flow:

```sh
cd /root/uboot/linux_recipe
git worktree add -b latest ../linux-build master
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

The image package installs board kernel artifacts under
`/usr/lib/linux-image-cv181x-sodoport/` and kernel modules under
`/lib/modules/<kernel-release>/`. The source package installs a compressed,
patched source tree under `/usr/src/`.
