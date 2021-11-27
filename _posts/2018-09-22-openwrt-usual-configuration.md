---
layout: post
title: OpenWrt 常用网络配置
categories: [Tutorial, OpenWrt]
tags: [OpenWrt]
seo:
  date_modified: 2020-04-22 12:43:02 +0800
---

文章记录了博主使用 OpenWrt 过程中多项常用网络功能的配置方法，包括 AP 模式、主机名设置、USB 联网等，以备后续之需。

## 切换为 AP 模式（Bridged AP）

一般场景，OpenWrt 路由器工作在「Router"」模式，包含了 NAT、拨号、DHCP、DNS 等功能。当想让一台无线 OpenWrt 设备仅作为无线接入点，而不提供路由功能，就需要让其工作在「Bridged AP」模式（俗称「无线AP」）。

![Bridged AP 模式网络拓扑](/assets/img/post/bridged.ap_v3.png)

按获取 IP 方式的不同，有两种方法让 OpenWrt 路由器工作在 AP 模式

### 方法 1 - 静态 IP 模式

`OpenWrt` - `Interfaces` - `LAN`

在 `Common Configuration` 中将 LAN 接口的静态 Ipv4 地址设置为与主路由 LAN 同网段的 IP，例如：主路由的 192.168.1.1，则这里可设置为 192.168.1.2。

在 `DHCP Server` 中勾选 `Disable DHCP for this interface.` 关闭本机 DHCP，由主路由进行 DHCP，以免冲突。

### 方法 2 - 自动获取 IP 模式

`OpenWrt` - `Interfaces` - `LAN`

在 `Common Configuration` 中修改 `Portocol` 协议为 `DHCP client`

方法二很简单，也比较灵活，但有一个缺点，即本机 IP 地址是随机的，要进入本机的管理面比较麻烦一下。因此最好结合主路由的静态 DHCP 规则或固定的局域网本地域名使用，参间本文 为设备配置本地域名本地域名 章节。

**注意：**博主建议使用方法 2，因为这样配置的路由器比较灵活：一次配置，多次使用，不需要根据主路由手动配置静态 IP。

参考：[Bridged AP OpenWrt Wiki](https://wiki.openwrt.org/doc/recipes/bridgedap)

## 为设备分配本地域名和静态 IP

我们知道主机名（hostname）可以代替 IP 地址进行访问，例如： `ping hostname` 、通过 `http://hostname/` 访问本地 Web 服务。

默认的，OpenWrt 会根据连接设备反馈的设备名配置其 DHCP 的 `Hostname` 记录，通过这些记录，IP 地址和 hostname 便一一关联起来。此外，OpenWrt 还为局域网配置了 `Local domain` ，默认后缀是 `.lan`（你可以修改为任意值，但注意不要和公网域名冲突）， 这种情况下 `Galaxy-S8.lan` 等价于 `Galaxy-S8`。

![OpenWrt DHCP list](/assets/img/post/2018/OpenWrt-DHCP-list.jpg)

如果对自动命名的主机名不满意或希望固定 IP，可以在 `Network` - `DHCP and DNS` - `Static Leases` 里自定义 `Hostname` 和对应的静态 IP。

**注意：**Chrome 浏览器需要在 hostname 的后面加一个 `/` 以转义关键词搜索；静态 IP 和 hostname 是绑定到 MAC 地址的，因此自定义 DHCP 静态记录时你需要指定设备的 MAC 地址。

## 使用 USB 绑定上网（USB RNDIS）

USB RNDIS 技术（协议），可以让操作系统通过 USB 设备虚拟出的网卡适配器连接到互联网，我们在 Android 手机上常见的 「USB 绑定/USB Tether」即使用此项技术。

OpenWrt 支持 RNDIS，因此也可以通过手机的 USB 绑定功能上网。方法如下

安装 `kmod-usb-net-rndis` 软件包

```sh
opkg install kmod-usb-net-rndis
```

手机通过 USB 数据线连接到 OpenWrt 设备的 USB 接口，然后打开手机的「`USB 绑定`」功能开关。

如果一切正常，OpenWrt 命令控制台可看到如下提示，同时 OpenWrt 会新增一个名为「`usb0`」的以太网适配器，表明 RNDIS 设备驱动成功

```
usb 1-4.1: new high-speed USB device number 58 using xhci_hcd
rndis_host 1-4.1:1.0 usb0: register 'rndis_host' at usb-0000:1b:00.0-4.1, RNDIS device, 02:05:xx:xx:xx:xx
```

**提醒：**如果你的 OpenWrt 运行在 ESXi 环境下，请参考我的这篇文章（TODO）为 OpenWrt 虚拟机添加 USB 设备。

然后 `OpenWrt` - `Network` - `Interfaces` - `Add new interface...` 新增一个名为 `TETHERING` 的接口，将其绑定到 `usb0`，协议选择 `DHCP client`。如图所示

![OpenWrt 添加 USB 接口](/assets/img/post/2018/openwrt-create-interface.png)

最后，`OpenWrt` - `Network` - `FireWall` 编辑 `wan` 域，将`TETHERING` 添加进来，就可以让你的 OpenWrt 使用手机网络上网了。

## 使用 USB 网卡上网（USB LTE MODEM）

如果你要让 OpenWrt 驱动一张 USB 4G 卡，需要安装以下依赖

```
kmod-usb-net-rndis, kmod-usb-net, kmod-usb2, usb-modeswitch, kmod-usb2-pci, kmod-usb-ohci-pci, kmod-usb-serial-option
```

然后参考 USB 绑定上网的配置方法

## 桥接多个网卡适配器到 LAN

如果你的 OpenWrt 运行在 ESXi 虚拟机下，你可能想让主机的多个网口（网卡适配器）绑定到 OpenWrt 的 LAN 接口，并且互相桥接（此时多个网口的关系相当于交换机）。配置很简单，如下

```sh
vim /etc/config/network 

## 仅单个网卡接入 LAN 的配置
config interface 'lan'
option proto 'static'
option ipaddr '192.168.1.1'
option netmask '255.255.255.0'
option ip6assign '60'
option _orig_ifname 'eth3'
option _orig_bridge 'false'
option ifname 'eth3'

## 多个网卡接入 LAN 并启用桥接的配置
config interface 'lan'
option proto 'static'
option ipaddr '192.168.1.1'
option netmask '255.255.255.0'
option ip6assign '60'
option _orig_ifname 'eth3'
option _orig_bridge 'true'
option type 'bridge'
option ifname 'eth0 eth2 eth3'
```

编辑完后执行以下命令使配置生效

```sh
/etc/init.d/network reload
```

实际效果如图所示

![桥接多个网卡到 LAN](/assets/img/post/2018/OpenWrt-bridge-multi-LAN-interfaces.png)
