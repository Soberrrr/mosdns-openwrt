#
# Copyright (C) 2024 OpenWrt Build Repository
#
# This is free software, licensed under the GNU General Public License v3.
#

include $(TOPDIR)/rules.mk

PKG_NAME:=mosdns
PKG_VERSION:=5.3.4
PKG_RELEASE:=1

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/IrineSistiana/mosdns.git
PKG_SOURCE_VERSION:=v$(PKG_VERSION)
PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz
PKG_MIRROR_HASH:=skip

PKG_LICENSE:=GPL-3.0-or-later
PKG_LICENSE_FILE:=LICENSE
PKG_MAINTAINER:=mosdns-openwrt build

PKG_BUILD_DEPENDS:=golang/host
PKG_BUILD_PARALLEL:=1
PKG_USE_MIPS16:=0

GO_PKG:=github.com/IrineSistiana/mosdns/v5
GO_PKG_LDFLAGS_X:=$(GO_PKG)/coremain.Version=v$(PKG_VERSION)

include $(INCLUDE_DIR)/package.mk
include $(TOPDIR)/feeds/packages/lang/golang/golang-package.mk

define Package/mosdns
  SECTION:=net
  CATEGORY:=Network
  SUBMENU:=IP Addresses and Names
  TITLE:=A plugin-based DNS forwarder/server
  URL:=https://github.com/IrineSistiana/mosdns
  DEPENDS:=$(GO_ARCH_DEPENDS) +ca-bundle
endef

define Package/mosdns/description
  mosdns is a "DNS forwarder/server" written in Go. Plugin-based,
  highly customizable. Useful for splitting DNS queries by domain,
  IP, response, etc.
endef

define Package/mosdns/conffiles
/etc/config/mosdns
/etc/mosdns/config.yaml
endef

define Package/mosdns/install
	$(call GoPackage/Package/Install/Bin,$(PKG_INSTALL_DIR))
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/bin/mosdns $(1)/usr/bin/mosdns

	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) ./files/mosdns.init $(1)/etc/init.d/mosdns

	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_CONF) ./files/mosdns.config $(1)/etc/config/mosdns

	$(INSTALL_DIR) $(1)/etc/mosdns
	$(INSTALL_CONF) ./files/config.yaml $(1)/etc/mosdns/config.yaml
endef

$(eval $(call GoBinPackage,mosdns))
$(eval $(call BuildPackage,mosdns))
