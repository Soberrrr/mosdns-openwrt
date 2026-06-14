# mosdns-openwrt

通过 GitHub Actions 自动将上游 [IrineSistiana/mosdns](https://github.com/IrineSistiana/mosdns) 编译为可在 OpenWrt 上安装的包。

- **目标 OpenWrt 版本**：`25.12.4`
- **目标平台**：`x86_64`（target `x86/64`）
- **mosdns 版本**：默认拉取最新 release，可在手动触发工作流时指定

## 使用方式

### 触发构建

1. **推送 tag**：在本仓库推送形如 `v5.3.4` 的 tag，将自动构建并发布 GitHub Release。
   ```sh
   git tag v5.3.4
   git push origin v5.3.4
   ```

2. **手动触发**：在 GitHub `Actions` 页面选择 *Build mosdns OpenWrt package* → *Run workflow*，可填入：
   - `mosdns_version`：mosdns 的 tag（如 `v5.3.4`）。留空则自动拉取最新。
   - `openwrt_version`：OpenWrt 版本，默认 `25.12.4`。
   - `target`：OpenWrt target，默认 `x86/64`。

### 下载产物

- **Tag 触发**：在仓库 Releases 页面下载附带的 `mosdns_*.ipk` / `mosdns-*.apk`。
- **手动触发**：在对应 Workflow Run 页面的 *Artifacts* 区域下载。

### 在 OpenWrt 上安装

OpenWrt 25.12 默认使用 APK，但部分镜像仍兼容 IPK。工作流会同时收集两种格式（取决于 SDK 实际产出）。

```sh
# APK 格式（推荐 25.12+）
apk add --allow-untrusted ./mosdns-*.apk

# IPK 格式（旧版兼容）
opkg install ./mosdns_*.ipk
```

启用并启动服务：

```sh
uci set mosdns.config.enabled=1
uci commit mosdns
/etc/init.d/mosdns enable
/etc/init.d/mosdns start
```

默认配置位于 `/etc/mosdns/config.yaml`，可按需修改。完整配置语法见 [mosdns wiki](https://irine-sistiana.gitbook.io/mosdns-wiki/)。

## 仓库结构

```
.
├── .github/workflows/build.yml   # GitHub Actions 工作流
├── package/mosdns/
│   ├── Makefile                  # OpenWrt 包定义
│   └── files/
│       ├── mosdns.init           # /etc/init.d/mosdns（procd 服务）
│       ├── mosdns.config         # /etc/config/mosdns（UCI 默认值）
│       └── config.yaml           # /etc/mosdns/config.yaml（mosdns 默认配置）
└── README.md
```

## 注意

- 本仓库仅打包 mosdns 主程序，不包含 LuCI Web 管理界面。如需 Web 配置界面，可参考 [sbwml/luci-app-mosdns](https://github.com/sbwml/luci-app-mosdns)。
- 上游若发布大版本（如 v6+），可能需要更新 `package/mosdns/Makefile` 中的 `GO_PKG` 路径（当前：`github.com/IrineSistiana/mosdns/v5`）。

## 许可

mosdns 上游使用 GPL-3.0-or-later。本仓库的打包脚本同样采用 GPL-3.0-or-later。
