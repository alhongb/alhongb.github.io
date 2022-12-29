---
title: OpenWrt 安装 Shadowsocks Server
categories: [Tutorial, Shadowsocks]
tags: [Shadowsocks]
layout: post
seo:
  date_modified: 2020-03-02 22:56:24 +0800
---

## 简介

在 OpenWrt 路由器上安装 server 版 Shadowsocks，使得客户端设备能够以 VPN 的形式远程连接到家庭局域网。

## 安装 Shadowsocks Server

### Step 1 - 安装 shadowsocks-libev-server 软件包

前往作者[项目主页](https://github.com/shadowsocks/openwrt-shadowsocks/releases)获取最新版本的 shadowsocks-libev-server 并安装，安装之前用如下命令查询自己路由器 CPU 所属的平台架构

```
opkg print-architecture
```

也可以使用软件源的方法安装，参考 [OpenWrt Shadowsocks 安装&配置指南](https://linhongbo.com/posts/shadowsocks-on-openwrt/#通过软件源安装) 添加软件源，然后执行以下命令安装

```
opkg install shadowsocks-libev-server
```

### Step 2 - 创建配置文件

```sh
vim /etc/shadowsocks-server.json

{
   "server":["[::0]","0.0.0.0"],
    "server_port":8888,
    "password":"xxxxxxx",
    "timeout":60,
    "method":"aes-128-gcm",
    "fast_open":true
}
```

### Step 3 - 创建开机启动脚本

出于安全考量，安装 sudo 来切换程序运行的用户和用户组

```sh
opkg install sudo
```

然后编辑 `/etc/rc.local` 启动脚本，增加一行启动命令：

```
## 在文件最后， exit 0 之前（如果有的话）加此行启动命令
sudo -u nobody -g nobody ss-server -u -c /etc/shadowsocks-server.json &
```

### Step 4 - 添加防火墙规则

点击 `Firewall - Traffic Rules` 添加一条防火墙规则，使得 WAN 侧（即 Internet）能够访问到 Shadowsocks 监听的端口

![openwrt shadowsocks server firewall](/assets/img/post/2020/openwrt-shadowsocks-server-firewall.png)

到此所有安装配置工作完成。

## 远程连接到内部网络

以 Android 客户端为例：路由规则根据需要选择`全局` 或 `绕过中国大陆`，让手机连接到路由器以访问内部网络，此外为了能正常解析域名（包括内网主机名），还需要将 `远程 DNS` 选项修改为 192.168.1.1（指向 OpenWrt 内建的 dnsmasq DNS 服务）并勾选 `使用 UDP DNS`。注意：适用于旧版客户端，新版客户端须参考以下文章更新。

2021-06 更新：新版本 Shadowsocks 客户端 DNS 按上述方法配置似乎只能解析内网主机，外网无法解析，原因不明，此时保持默认的 dns.google 即可修复（但无法解析内网）。

2022-02 更新：造成上述异常的实际原因是新版本 Shadowsocks Android 客户端不再支持使用 UDP 来请求远程 DNS 服务器，只会用 TCP 来请求，那么当远程 DNS 设置为路由器上的 dnsmasq 时，虽然 dnsmasq 支持 TCP，但若其配套的上游 DNS 服务器（例如 ChinaDNS）不支持 TCP（可以通过 `netstat -nalp` 确定这一点），则客户端的 TCP DNS 请求在转发给这些上游 DNS 时就会失败（dnsmasq 日志显示 `REFUSED`）。~~终极解决办法（不稳定）：在 dnsmasq 的 `DNS forwardings` 选项中额外加入支持 TCP 请求的 `8.8.8.8#53` 作为备用 DNS~~（多个上游 DNS 存在时，dnsmasq 会按次序请求，仅当请求失败时使用后续的 DNS，因此此操作几乎没有副作用）。参考：[dnsmasq 一直返回 REFUSED](https://github.com/aa65535/openwrt-dns-forwarder/issues/30#issue-687825881)

![Shadowsocks Clent Config](/assets/img/post/2020/client-configuration-for-openwrt-shadowsocks-server.jpg)
