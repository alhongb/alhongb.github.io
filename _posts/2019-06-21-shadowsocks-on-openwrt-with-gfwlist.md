---
layout: post
title: OpenWrt Shadowsocks 使用 GFWList 路由规则
categories: [Tutorial, Shadowsocks]
tags: [Shadowsocks, OpenWrt]
seo:
  date_modified: 2020-03-02 22:39:56 +0800
---

## 介绍

在 [《OpenWrt Shadowsocks 安装&配置指南》]({% post_url 2018-07-21-shadowsocks-on-openwrt %})一文中博主详细介绍了使用 CHNRoutes 规则翻墙方案，本文进一步说明如何在其基础上切换为 GFWList 规则，即仅在 GFWList 中流量的走 Shadowsocks 代理。方案的基本思路是基于 GFWList IP 地址列表构建 iptables (防火墙) 规则，将目标 IP 的流量转发至 Shadowsocks。

**注意：**你首先需要完成前述文章的配置内容

### Step 1 - 准备 GFWList ipset 集

ipset 是一个iptables 的辅助工具，能够轻松愉快地创建和维护一组IP地址。本文通过配置 dnsmasq ，命令其将 GFWList 列表中的域名一一解析成 IP 地址，并记录为一个 ipset 集。

首先，我们创建一个专门存放 dnsmasq.d 自定义配置文件的目录

```sh
mkdir /etc/dnsmasq.d
```

接着，指定 dnsmasq 加载该目录

- OpenWrt

修改 /etc/dnsmasq.conf，最后加一行：

```sh
vim /etc/dnsmasq.conf

...
conf-dir=/etc/dnsmasq.d
```

- LEDE

执行以下命令获取当前的配置状态

```sh
uci get dhcp.@dnsmasq[0].confdir
```

如果返回值为 `uci: Entry not found` 或不是 `/etc/dnsmasq.d` ，则执行：

```sh
uci add_list dhcp.@dnsmasq[0].confdir=/etc/dnsmasq.d
uci commit dhcp
```

下载已经整理好的规则文件到 dnsmasq.d 目录

```sh
cd /etc/dnsmasq.d
wget https://cokebar.github.io/gfwlist2dnsmasq/dnsmasq_gfwlist_ipset.conf
```

配置文件的语法是用 `server` 指定某 GFW 域名用某 DNS 解析，然后将解析结果 `ipset` 到一个名为 gfwlist 的地址集中。片段如下：

```
dnsmasq_gfwlist_ipset.conf
...
server=/030buy.com/127.0.0.1#5353
ipset=/030buy.com/gfwlist
```

在 `Network - Firewall - [Custom Rules](http://lede/cgi-bin/luci/admin/network/firewall/custom)` 中添加一条 `ipset create gfwlist hash:ip`，让 iptables 启动时创建 `gfwlist` ipset

重启 dnsmasq，完成所有配置。

```sh
/etc/init.d/dnsmasq restart
```

最后，为了测试 `gfwlist`  是否成功创建，你可以先访问一些存在于 GFWlist 中的网站，然后执行

```sh
ipset list gfwlist
```

检查 `gfwlist` 这个 ipset 是否存在且包含解析的 IP 地址。

### Step 2 - 将 Shadowsocks 路由规则切换到 GFWlist

参考 [luci-app-shadowsocks/wiki/GfwList-Support](https://github.com/shadowsocks/luci-app-shadowsocks/wiki/GfwList-Support) 将 Shadowsocks 的 `access control/访问控制` 规则修改为：

> 进入 Luci 界面 -> 访问控制 -> 外网区域 「被忽略IP列表」 选择留空(/dev/null) 「额外被忽略IP」 设置为 `0.0.0.0/1` 和 `128.0.0.0/1`

即忽略所有 IP 的流量

然后执行 iptables 命令，将匹配 `gfwlist` 的流量转发至 Shadowsocks

```sh
iptables -t nat -I SS_SPEC_WAN_AC 1 -m set --match-set gfwlist dst -j SS_SPEC_WAN_FW
```

**注意：**每次重启 Shadowsocks 后都需要运行一次此命令。如果想随 Shadowsocks 启动时运行此命令，编辑其启动脚本：

```sh
vim /etc/init.d/shadowsocks

...
start() {
    pidof ss-redir ss-local ss-tunnel >/dev/null && return 0
    mkdir -p /var/run /var/etc
    ss_redir && rules
    ss_local
    ss_tunnel

    #增加此行
    iptables -t nat -I SS_SPEC_WAN_AC 1 -m set --match-set gfwlist dst -j SS_SPEC_WAN_FW

} 
...
``` 

至此，Shadosocks 路由规则完成从 CH NRoutes 切换到 GFWlist ，如果要反向操作，将 Step 2 的修改复原即可。
