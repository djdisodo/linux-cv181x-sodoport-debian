# Recipe-only Debian packaging for Linux 7.0

`linux_recipe/` is a packaging-only Git repository for a Milk-V Duo 256M
kernel package built from upstream Linux `7.0`.

The repository intentionally does not contain unpacked upstream sources. The
expected workflow is:

1. Keep the recipe on `master`.
2. Import upstream tarballs into a generated `latest` branch with
   `gbp import-orig`.
3. Maintain board-specific DTS patches with `dpkg-source --commit`.

The package scaffold is intentionally incomplete until we add:

- `debian/config/sg2002-milkv-duo256m.config`
- DTS patches in `debian/patches/`

Suggested local flow:

```sh
cd /root/uboot/linux_recipe
git worktree add -b latest ../linux-build master
cd ../linux-build
version=$(dpkg-parsechangelog --show-field Version | sed 's/-[^-]*$//')
uscan --check-dirname-level 0 --download-current-version --rename --destdir ..
gbp import-orig --no-interactive --debian-branch=latest \
  --upstream-branch=upstream/latest --upstream-version "$version" \
  ../linux-sg2002-milkv-duo256m_${version}.orig.tar.xz
dpkg-buildpackage -us -uc -b -a riscv64
```

Once the upstream tree is imported, create or refresh patches with:

```sh
dpkg-source --commit
```

The binary package is intended to install board kernel artifacts under
`/usr/lib/linux-image-sg2002-milkv-duo256m/`.
