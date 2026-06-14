# mosdns-openwrt

OpenWrt package recipe for [IrineSistiana/mosdns](https://github.com/IrineSistiana/mosdns) — a plugin-based DNS forwarder/server written in Go.

This repo contains only the OpenWrt packaging (Makefile, init script, UCI config, default `config.yaml`) plus a GitHub Actions pipeline that builds `.ipk` / `.apk` artifacts using the official OpenWrt SDK. No mosdns source lives here.

## Downloads

Pre-built packages for tagged releases are published to the [Releases page](https://github.com/Soberrrr/mosdns-openwrt/releases).

Each release lists the upstream mosdns version, OpenWrt release, and target architecture in its notes.

## Build via GitHub Actions

The `Build mosdns OpenWrt package` workflow can be triggered two ways:

- **Tag push** — push a `vX.Y.Z` tag; the workflow builds and creates a GitHub Release with the artifacts.
- **Manual dispatch** — run the workflow from the Actions tab with optional inputs:
  - `mosdns_version` — e.g. `v5.3.4`. Empty = fetch latest upstream release.
  - `openwrt_version` — default `latest` (resolves to the newest non-prerelease tag from `openwrt/openwrt` via the GitHub API). Pass an explicit version like `25.12.4` or `24.10.7` to pin.
  - `target` — default `x86/64` (e.g. `ramips/mt7621`, `aarch64_cortex-a53`, etc.).

Tag-driven release builds always use the pinned `OPENWRT_VERSION_DEFAULT` declared in `.github/workflows/build.yml` (currently `25.12.4`) so the same mosdns tag rebuilds reproducibly. Bump that env when you intentionally move the release line forward.

The SDK tarball name (which encodes the toolchain GCC version) is resolved at build time from the target's `sha256sums` index, so new OpenWrt releases with newer toolchains work without workflow edits.

Note: SDK 25.x produces `.apk`; older SDKs produce `.ipk`. The workflow detects and uploads whichever format is built.

## Build locally

Requires the OpenWrt SDK matching your target. Roughly:

```sh
# 1. Download and extract the OpenWrt SDK for your target
curl -fL -O https://downloads.openwrt.org/releases/25.12.4/targets/x86/64/openwrt-sdk-25.12.4-x86-64_gcc-14.3.0_musl.Linux-x86_64.tar.zst
mkdir sdk && tar -I zstd -xf openwrt-sdk-*.tar.zst -C sdk --strip-components=1

# 2. Update feeds
cd sdk
./scripts/feeds update -a
./scripts/feeds install -a

# 3. Drop this package in
cp -r ../package/mosdns package/mosdns

# 4. Configure and build
make defconfig
echo 'CONFIG_PACKAGE_mosdns=m' >> .config
make defconfig
make package/mosdns/compile -j"$(nproc)" V=s

# 5. Artifact
find bin/packages -name 'mosdns*.ipk' -o -name 'mosdns*.apk'
```

## Install on the device

```sh
# apk-based OpenWrt (24.10+/25.x default)
apk add ./mosdns_*.apk

# opkg-based OpenWrt
opkg install ./mosdns_*.ipk
```

After install, the service is **disabled by default**. Enable and start:

```sh
uci set mosdns.config.enabled=1
uci commit mosdns
/etc/init.d/mosdns enable
/etc/init.d/mosdns start
```

## What gets installed

| Path | Purpose |
| --- | --- |
| `/usr/bin/mosdns` | Binary |
| `/etc/init.d/mosdns` | procd init script |
| `/etc/config/mosdns` | UCI config (conffile, preserved on upgrade) |
| `/etc/mosdns/config.yaml` | mosdns config (conffile, preserved on upgrade) |
| `/var/lib/mosdns` | Working directory, created on service start |
| `/var/log/mosdns.log` | Log path (when configured) |

## UCI options

`/etc/config/mosdns`:

| Option | Default | Notes |
| --- | --- | --- |
| `enabled` | `0` | Set to `1` to allow the init script to start the service |
| `config_file` | `/etc/mosdns/config.yaml` | Passed to `mosdns start --config` |
| `log_file` | `/var/log/mosdns.log` | Parent dir is created on start |
| `working_dir` | `/var/lib/mosdns` | Passed to `mosdns start --dir`; used by plugins for cache/data |

## Default `config.yaml`

The shipped default forwards all queries over DoH to Cloudflare and Google, listening on UDP/TCP `:5335`. Replace it with your own configuration — see the upstream [mosdns documentation](https://irine-sistiana.gitbook.io/mosdns-wiki/) for plugin reference.

To use mosdns as the system resolver, point dnsmasq at it (e.g. set `server=127.0.0.1#5335` and `no-resolv`).

## Versioning

`PKG_VERSION` in `package/mosdns/Makefile` pins the upstream mosdns tag (`v$(PKG_VERSION)`). CI overrides it with the resolved version at build time, so manual edits are only needed when developing locally. Bump `PKG_RELEASE` for packaging-only changes.

## License

Packaging files: GPL-3.0-or-later (matching upstream mosdns).
