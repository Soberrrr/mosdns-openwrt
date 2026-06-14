# AGENTS.md

## Repo purpose

Not the mosdns source. This repo only ships an **OpenWrt package recipe** for upstream `IrineSistiana/mosdns` and a GitHub Actions pipeline that builds it inside the OpenWrt SDK. There is no Go code here; `go build` locally is meaningless.

## Layout

- `package/mosdns/Makefile` — OpenWrt package Makefile (uses `golang-package.mk`, `GoBinPackage`). Pins upstream mosdns via `PKG_VERSION` + `PKG_SOURCE_VERSION:=v$(PKG_VERSION)`.
- `package/mosdns/files/` — installed onto the device:
  - `mosdns.init` — procd init script (`/etc/init.d/mosdns`), runs `mosdns start --config ... --dir ...`.
  - `mosdns.config` — UCI defaults at `/etc/config/mosdns` (read by the init script via `config_get`: `enabled`, `config_file`, `log_file`, `working_dir`).
  - `config.yaml` — default mosdns config at `/etc/mosdns/config.yaml`.
  Both `/etc/config/mosdns` and `/etc/mosdns/config.yaml` are declared `conffiles` — preserved across upgrades.
- `.github/workflows/build.yml` — the only build path that actually works.

## Build / verify

There is no local `make` here. Building requires the OpenWrt SDK. Two options:

1. **Use CI** — push a `v*` tag (triggers release; always uses the pinned `OPENWRT_VERSION_DEFAULT` from `build.yml` for reproducibility), or run the `Build mosdns OpenWrt package` workflow via `workflow_dispatch` with optional `mosdns_version`, `openwrt_version` (default `latest` — resolved at runtime from `openwrt/openwrt` GitHub releases, fall back to `OPENWRT_VERSION_DEFAULT` on API failure), `target` (default `x86/64`).
2. **Local repro** — mirror the workflow: download `openwrt-sdk-<ver>-<target>_gcc-14.3.0_musl.Linux-x86_64.tar.zst`, extract, `./scripts/feeds update -a && ./scripts/feeds install -a`, copy `package/mosdns` into `sdk/package/mosdns`, `make defconfig`, append `CONFIG_PACKAGE_mosdns=m`, `make defconfig`, then `make package/mosdns/compile -j$(nproc) V=s`.

Output is `.ipk` or `.apk` depending on SDK version (the workflow auto-detects). 25.x SDKs produce `.apk`.

## Version bumping

The CI overrides `PKG_VERSION` in the Makefile at build time using the resolved upstream mosdns tag (`sed -i 's/^PKG_VERSION:=.*/.../'`). When updating manually:

- Edit only `PKG_VERSION` in `package/mosdns/Makefile`. `PKG_SOURCE_VERSION` derives from it (`v$(PKG_VERSION)`).
- Bump `PKG_RELEASE` if packaging-only changes ship without an upstream version change.
- `PKG_MIRROR_HASH:=skip` is intentional (git source, no tarball hash pinning).

Tagging `vX.Y.Z` triggers the release job and publishes artifacts to GitHub Releases.

## Gotchas

- `GO_PKG:=github.com/IrineSistiana/mosdns/v5` — the `/v5` matters; don't drop it on a major bump.
- `GO_PKG_LDFLAGS_X` injects the version into `coremain.Version`; keep this in sync if upstream renames the variable.
- Init script exits early unless UCI `mosdns.config.enabled=1`. After install the service is disabled by default.
- `working_dir` defaults to `/var/lib/mosdns` and is created by the init script — referenced by mosdns plugins for cache/data files.
- `PKG_USE_MIPS16:=0` is required for Go packages on MIPS targets; do not remove.
- Workflow needs `jq` (installed in CI) to resolve "latest" mosdns release when `mosdns_version` input is empty.
- SDK tarball name (including GCC version, e.g. `gcc-14.3.0`) is resolved at build time by parsing `releases/<ver>/targets/<target>/sha256sums`. If upstream changes the tarball naming scheme, update the `grep -oE 'openwrt-sdk-...\.tar\.zst'` regex in `Resolve build parameters`.
- "latest" OpenWrt resolution uses `/repos/openwrt/openwrt/releases?per_page=30` and filters non-prerelease `vX.Y.Z` tags; tag-driven release builds bypass this and use the env default for reproducibility.

## When editing

- Changes to `files/*` must remain shell/UCI/YAML compatible with OpenWrt 24.x+ (procd, ash — not bash).
- Don't add `cd` chains or bashisms to `mosdns.init`; busybox ash only.
- If you add a new conffile, register it in the `Package/mosdns/conffiles` block, otherwise upgrades will overwrite user edits.
