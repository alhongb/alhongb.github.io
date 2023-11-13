---
layout: post
title: OpenWrt Shadowsocks 安装&配置指南
categories: [Tutorial, Shadowsocks]
tags: [OpenWrt, Shadowsocks]
---

## 前言

这是一篇 OpenWrt 19（19.07.4 版本验证）下安装 Shadowsocks 的教程。由于 OpenWrt 从 22.03.0 开始，使用 nftables 代替了 iptables 来配置 Linux 的网络过滤规则，而本文中部分软件截止文章更新时仍未对此做兼容，因此**下文所述方案已经无法在最新版本的 OpenWrt 正常工作**，因此如果你是新版本的 OpenWrt，请绕行或关注 [luci-app-shadowsocks](https://github.com/shadowsocks/luci-app-shadowsocks/issues/301) 是否更新了新的版本。

同时，如果你只使用机场，大部分机场使用 Clash 就可以很好地工作，因此博主强烈建议在 OpenWrt 下使用 ShellCrash，详细安装配置过程参考[这篇文章](https://linhongbo.com/posts/how-to-use-clash-on-openwrt/)。

## 简介

文章先介绍了整体软件栈和通信原理，然后详细说明了各个软件的安装、配置过程，最终可搭建一个路由规则是国内 IP 直连、其余 IP 走透明代理的翻墙路由器（即 CHNRoutes 黑名单模式）。

要使用 GFWList 路由模式（即仅被 GFW 封锁的流量走代理），请参阅本站另外一篇文章：[《OpenWrt Shadowsocks GFWlist 配置教程》]({% post_url 2019-06-21-shadowsocks-on-openwrt-with-gfwlist %})。

提示：推荐使用 CHNRoutes 路由规则，效果稳定且能够避免部分 GFWList 遗漏的网站无法访问，对于外网访问也有一定的加速效果（Shadowsocks 服务端网络连接性能优良的前提下）。

## 软件逻辑架构

Shadowsocks for OpenWrt 是基于 shadowsocks-libev 移植的，包含 **ss-local、ss-redir** 和 **ss-tunnel** 三个组件。此外，一个完整的 Shadowsocks 透明代理解决方案还包括 ChinaDNS 和远程 Shadowsocks 服务器等配套服务，整体逻辑架构图如下

![OpenWrt Shdowsocks 逻辑架构图](/assets/img/post/2018/architecture-of-openwrt-shadowsocks-transparent-proxy.png)

其中，**ss-redir** 负责将 OpenWrt 的 TCP/UDP 出口流量透明地转发至境外 shadowsocks 代理服务器；**ss-local** 是本地 SOCKS5 代理服务器，可额外地为浏览器等客户端应用提供 SOCKS5 代理服务；**Dnsmaq** 是 OpenWrt 的默认 DNS 转发服务，本方案中负责接收来自局域网的 DNS 请求后转发给 ChinaDNS 处理；**ChinaDNS** 是一个开源的防 DNS 污染解决方案，它通过 **ss-tunnel** 转发 DNS 请求到墙外服务器，从而获取无污染的解析结果。

## Step 1 - 安装&配置 Shadowsocks

### 安装 Shadowsocks

先安装 shadowsocks-libev  `UDP-Relay` 功能的依赖包 `iptables-mod-tproxy`

```sh
opkg update
opkg install iptables-mod-tproxy 
```

由于 OpenWrt 内建的 wget 不支持 TLS，无法下载 HTTPS 网站上的软件包，因此还要安装好整版的 wget 和 CA 证书软件，前者负责下载链接，后者提供 HTTPS 连接所需的根证书：

```sh
opkg install wget ca-certificates
```

最后，正式开始安装 `shadowsocks-libev` 以及 `luci-app-shadowsocks`。可使用离线下载预编译包和软件源两种安装方式，读者可根据本地环境不同选择。

- **使用作者的软件源安装 Shadowsocks（推荐方法）**

添加软件源公钥

```sh
wget http://openwrt-dist.sourceforge.net/packages/openwrt-dist.pub
opkg-key add openwrt-dist.pub
```

添加软件源到配置文件，**注意**务必将 *x86_64* 替换为你自己硬件的 CPU 架构名，可用 `opkg print-architecture` 命令查询。

```sh
vim /etc/opkg/customfeeds.conf

src/gz openwrt_dist http://openwrt-dist.sourceforge.net/packages/base/x86_64
src/gz openwrt_dist_luci http://openwrt-dist.sourceforge.net/packages/luci
```

安装 shadowsocks-libev、luci-app-shadowsocks

```sh
opkg update
opkg install shadowsocks-libev
opkg install luci-app-shadowsocks
```
- 使用 OpenWrt 自带软件源安装 Shadowsocks（不推荐）

最新的 OpenWrt （例如 19.07） 已经已经自带 Shadowsocks 软件包，可参照 https://openwrt.org/docs/guide-user/services/proxy/shadowsocks 的说明来安装，但是自带软件源的 Shadowsocks 版本太低，不太推荐。

安装如下软件包

```sh
opkg install shadowsocks-libev-ss-local shadowsocks-libev-ss-redir shadowsocks-libev-ss-rules shadowsocks-libev-ss-tunnel
```

如果需要 Luci Web 界面还要安装

```sh
opkg install luci-app-shadowsocks-libev
```

- **手动下载预编译包安装 Shadowsocks（备用方法）**

前往作者 github 主页获取最新的预编译版本 [shadowsocks-libev](https://github.com/shadowsocks/openwrt-shadowsocks/releases) 和 [luci-app-shadowsocks](https://github.com/shadowsocks/luci-app-shadowsocks/releases) UI 界面。注意，选择 `current` 目录下匹配你硬件架构的版本。使用以下命令查询你的路由器硬件架构，例如博主是 `x86_64`。

```sh
opkg print-architecture
```

获取链接后，下载并安装上述软件包

```sh
cd /tmp
wget https://xxx.ipk
opkg install xxx.ipk
```

### 配置 Shadowsocks

- Servers Manage
    管理 Shadowsocks 服务器节点。按照实际情况填写你的 Shadowsocks 代理服务器信息即可。
    **注意**：如果要开启 `TCP Fast Open` 选项，需要修改 `sysctl.conf` 添加一行net.ipv4.tcp_fastopen = 3，然后使之生效。命令如下
    ```sh
    echo "net.ipv4.tcp_fastopen = 3" >> /etc/sysctl.conf
    sysctl -p
    ```
- General Settings
    使能 `Transparent Proxy` 和 `Port Forward`，其中 `UDP-Relay` 是 UDP 转发功能，这里要将其开启，其余配置项保持默认即可。

- Access Control
    `Bypassed IP List` 选择 `ChinaDNS CHNRoute`（ChinaDNS 安装后，IP 路由文件的路径，例如 `/etc/chinadns-ng/chinalist.txt`）或自定义 CHNRoute 文件地址（参考[安装 ChinaDNS](#安装-chinadns) 中关于 `/etc/chinadns_chnroute.txt` 的描述）
    Shadowsocks 的访问规则控制。CHNRoute 是一种合适规则是国外网站走 Shadowsocks 代理而国内网站直连，这样通常还可以加速国外网站，相应的，也可以选择 GFWRoute。

## Step 2 安装&配置 ChinaDNS

国内运营商网络 DNS 污染严重，导致大量境外域名无法正确解析，而 shadowsocks-libev 本身并没有解决 DNS 污染问题，需要配合 [ChinaDNS](https://github.com/aa65535/openwrt-chinadns) 组件来解决。

另外，也可以采用重构优化的 [ChinaDNS-NG](#方案二---采用-chinadns-ng) 来替代。二者主要区别：

- ChinaDNS 安装简单、使用稳定可靠，但最近一次更新是 2015 年；
- ChinaDNS-NG 是新近的项目，需要自行编译，稳定性还需要验证；
- ChinaDNS 无法显式定义可信 DNS 和 国内 DNS，而是根据 IP 地址自动判断；
- ChinaDNS 也无法显式定义具体域名的解析行为，譬如指定域名使用可信 DNS（或国内 DNS）解析。

读者可自行决定选择哪一个方案。

### 方案一 - 采用 ChinaDNS

#### 安装 ChinaDNS

ChinaDNS 解决 DNS 污染的思路如下：

> ChinaDNS 分国内 DNS 和可信 DNS。ChinaDNS 会同时向国内 DNS 和可信 DNS 发请求，如果可信 DNS 先返回，则采用可信 DNS 的数据；如果国内 DNS 先返回，又分两种情况，返回的数据是国内的 IP 则采用，否则丢弃并转而采用可信 DNS 的结果。
> 

安装方法同样有两种：

1. 用软件源安装
    如果先前安装 shadowsocks-libev 时已添加了 openwrt-dist 源 ，可直接命令行安装 ChinaDNS  
    ```sh
    opkg install ChinaDNS
    opkg install luci-app-chinadns
    ```

2. 下载预编译版本安装
    前往项目主页获取最新的预编译版本 [ChinaDNS](https://github.com/aa65535/openwrt-chinadns/releases) （选择 `current` 目录下特定硬件平台版本） 和 [chinadns-luci-app](https://github.com/aa65535/openwrt-dist-luci/releases) 并安装：  
    ```sh
    cd /tmp
    wget https://xxx.ipk
    opkg install xxx.ipk
    ```

安装完成后立即更新 ChinaDNS 的国内 IP 路由表 `/etc/chinadns_chnroute.txt`，这里分别提供 apnic 和国内的 [ipip.net](https://github.com/17mon/china_ip_list) 整理的路由表更新命令，推荐注重性能的使用后者：

- apnic
    ```sh
    wget -O /tmp/delegated-apnic-latest 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' && awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' /tmp/delegated-apnic-latest > /etc/chinadns_chnroute.txt
    ```

- ipip.net
    ```
    wget https://raw.githubusercontent.com/17mon/china_ip_list/master/china_ip_list.txt -O /tmp/china_ip_list.txt && mv /tmp/china_ip_list.txt /etc/chinadns_chnroute.txt
    ```

使用 `crontab -e` 命令编辑 cron 任务计划，每月（1 号凌晨 3 点）更新 `chinadns_chnroute.txt`，编辑内容如下（apnic 和 ipip.net 择一）：

```
#For apnic
0 3 1 * *    wget http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest -O /tmp/delegated-apnic-latest && awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' /tmp/delegated-apnic-latest > /etc/chinadns_chnroute.txt

#For ipip.net
0 3 1 * *    wget https://raw.githubusercontent.com/17mon/china_ip_list/master/china_ip_list.txt -O /tmp/china_ip_list.txt && mv /tmp/china_ip_list.txt /etc/chinadns_chnroute.txt
```

编辑完成后启用 cron 程序：

```sh
/etc/init.d/cron start
/etc/init.d/cron enable
```

验证 crontab 任务是否正确执行：

```sh
logread | grep crond
```

**注意：**不要直接使用 `vim` 编辑 `/etc/crontabs/root` 文件；本文提供的更新命令已经过优化，网络连接失败的情况下不会覆写错误信息到 `/etc/chinadns_chnroute.txt` 避免造成异常。

安装好后如图所示（忽略 vlmcsd ）：

![](/assets/img/post/2018/shadowsocks-chinadns.png)

#### 配置 ChinaDNS

勾选 `Also filter results inside China from foreign DNS servers`、将上游 DNS 修改为

`114.114.114.114,127.0.0.1:5300`

其中国内 IP DNS 用于解析国内域名，建议使用运营商提供的DNS地址（注：因为有时114DNS速度慢于远程DNS，特别是远程DNS转发性能特别好时，而运营商DNS则一般不会发生这种情况。通过查看 /etc/resolv.conf 得到本地运营商DNS）；`127.0.0.1:5300` 为 `ss-tunnel` 提供的 DNS 端口转发服务，负责DNS的远程服务器解析。最后，启动 ChinaDNS。

如图所示

![ChinaDNS 配置示例](/assets/img/post/2018/chinadns-configuration.png) 

这里结合[代码](https://github.com/shadowsocks/ChinaDNS/blob/1.3.2/src/chinadns.c#L760)深入讲解一下 ChinaDNS 的上游 DNS（`-s` 参数）工作模型：

```c
  int dns_is_chn = 0;
  int dns_is_foreign = 0;
  if (chnroute_file && (dns_servers_len > 1)) {
    dns_is_chn = test_ip_in_list(dns_addr, &chnroute_list);
    dns_is_foreign = !dns_is_chn;
  }
```
{: file='chinadns.c'}

当配置了多个上游 DNS 时，ChinaDNS 简单通过判断 DNS 的 IP 地址是否在国内来定义为「国内 DNS」， 否则一律为「国外 DNS」，这个隐式逻辑虽然简单易用但会在复杂配置场景带来问题，典型的：当「国内 DNS」是用 Stubby 搭建的本地代理 DNS 时，会将 Stubby 误判成「国外 DNS」，导致非预期的行为。

### 方案二 - 采用 ChinaDNS-NG

[ChinaDNS-NG 项目](https://github.com/zfl9/chinadns-ng) 目前并没有提供预编译包，需要使用 openwrt sdk 来手动编译，比较麻烦，可以使用下面的预编译包来安装。

#### 使用 ChinaDNS-NG 预编译包安装

可以在 (op.supes.top)[https://op.supes.top/packages/mips_24kc/] 下载安装 ChinaDNS-NG 预编译包，包括一个可执行文件包和一个 luci 包。

#### 编译 ChinaDNS-NG 安装

出于方便本文使用 docker 来创造 openwrt 编译环境。

首先，进入 [docker hub](https://hub.docker.com/r/openwrtorg/sdk/tags) 找到匹配你的 openwrt 软硬件（可用 `opkg print-architecture` 查到硬件架构）的 SDK 镜像，然后使用 docker pull 命令拉取，例如：

```shell
docker pull openwrtorg/sdk:mips_24kc-19.07.4
```

接着启动容器：

```shell
docker run --rm -v "$(pwd)"/bin/:/home/build/openwrt/bin -it openwrtorg/sdk:mips_24kc-19.07.4
```

在容器内执行以下命令来编译 ChinaDNS-NG：

```shell
#获取源码
git clone https://github.com/pexcn/openwrt-chinadns-ng.git package/chinadns-ng

#选中 Network -> chinadns-ng （由空变成 M：单独打包，不编入系统镜像，或 *：打包并编入镜像）
make menuconfig

#编译 chinadns-ng
make package/chinadns-ng/{clean,compile} V=s
```

在容器内执行以下命令来编译 [ChinaDNS-NG LuCI APP](https://github.com/pexcn/openwrt-chinadns-ng/tree/luci)

```shell
#获取源码
git clone -b luci https://github.com/pexcn/openwrt-chinadns-ng.git package/luci-app-chinadns-ng

#安装 luci 依赖，否则 meke menuconfig 无 LuCI 子项可选
./scripts/feeds update -a
./scripts/feeds install luci

#选中 LuCI -> Applications -> luci-app-chinadns-ng（由空变成 M 或 *）
make menuconfig

#编译 luci-app-chinadns-ng
make package/luci-app-chinadns-ng/{clean,compile} V=s
```

最后在主机的 `"$(pwd)"/bin` 目录，就能找到编译结果了。

将将编译出来的 ChinaDNS-NG 和 ChinaDNS-NG LuCI APP 的 ipk 包安装到 openwrt，这里有个坑，由于  ChinaDNS-NG LuCI APP 安装时会再次释放 ChinaDNS-NG 配置文件到系统，若提示文件冲突导致安装失败，可以手动删除冲突的配置文件后再次安装。

#### 配置 ChinaDNS-NG

安装完毕，进入 ChinaDNS-NG 的 Web 界面，修改 `Trusted DNS Servers` 为 `127.0.0.1#5300` ，按你的喜好修改 `China DNS Servers`（这里如果希望使用 DNS over TLS 的，可参考博主的另外一篇[文章](https://linhongbo.com/posts/dns-over-tls-with-stubby-on-openwrt/)），最后启动 ChinaDNS-NG 即可。

ChinaDNS-NG 高级配置：

- `Enable the Fair_Mode`：当可信 DNS 可能先于国内 DNS 返回时建议开启。当可信 DNS 先于国内 DNS 返回时（极少数情况如此），原版 ChinaDNS 是直接采用可信 DNS 的结果，导致国内域名的解析被可信 DNS 抢答，如果该国内域名又正好有境外服务器，解析结果就非最优线路。该选项就是解决这一问题：当可信 DNS 先返回时，ChinaDNS-NG 并不立即采用，而是等待国内 DNS 的结果，随后如果国内 DNS 返回国内 IP，则采用，否则采用可信 DNS 的结果。这个选项大部分情况会按预期运行，但不排除[极特殊情形](https://github.com/zfl9/chinadns-ng/issues/11)下会有异常。
- `Black List` 和 `White List`：分别对应「强制走可信 DNS 的域名列表」 和「强制走国内 DNS 的域名列表」。

## Step 3 - 配置 Dnsmasq

OpenWrt 管理面 `Network` -> `DHCP and DNS`

`DNS forwardings` 修改为 `127.0.0.1#5353` 即 ChinaDNS 监听的端口；勾选 `Ignore resolve file`

提示：`Ignore resolve file` 指的是忽略 `/etc/resolv.conf` 中的 DNS ，即 WAN 口的 DNS，默认是电信运营商分配的 DNS；要恢复默认选项（取消勾选），在 `Resolve file` 一栏填入 `/tmp/resolv.conf.auto` 或者 `/tmp/resolv.conf.d/resolv.conf.auto` （新版 OpenWrt）

## Step 4（可选） - 让路由器自身也翻墙

到此步骤为止，之前的配置已足以让接入局域网的设备正常翻墙，但为了让路由器自身发起的连接也能够翻墙成功，需要将 WAN 口默认使用的运营商 DNS 修改为 ChinaDNS

如图所示，在 `Network - Interfaces - WAN、WWAN（无线接入时） - Advanced Settings` 中去掉 `Use DNS servers advertised by peer` ，并在配置栏中填入 `127.0.0.1`

![OpenWrt WAN DNS Configuration](/assets/img/post/2018/wan-advanced-settings.jpg)

至此，一切准备就绪，Enjoy yourself! :)


## 常见问题处理

- **Shadowsocks 无法启动**

重启路由器。如果仍未解决，使用 `logread` 命令查看异常日志

- **Luci 界面无法打开**

点击安装好 Shadowsocks 或 ChinaDNS 的 Luci APP 报错

```
/usr/lib/lua/luci/dispatcher.lua:1334: module 'luci.cbi' not found:
	no field package.preload['luci.cbi']
	no file './luci/cbi.lua'
	no file '/usr/share/lua/luci/cbi.lua'
	no file '/usr/share/lua/luci/cbi/init.lua'
	no file '/usr/lib/lua/luci/cbi.lua'
	no file '/usr/lib/lua/luci/cbi/init.lua'
	no file './luci/cbi.so'
	no file '/usr/lib/lua/luci/cbi.so'
	no file '/usr/lib/lua/loadall.so'
	no file './luci.so'
	no file '/usr/lib/lua/luci.so'
	no file '/usr/lib/lua/loadall.so'
stack traceback:
	[C]: in function 'require'
	/usr/lib/lua/luci/dispatcher.lua:1334: in function '_cbi'
	/usr/lib/lua/luci/dispatcher.lua:1023: in function 'dispatch'
	/usr/lib/lua/luci/dispatcher.lua:999: in function 'dispatch'
	/usr/lib/lua/luci/dispatcher.lua:478: in function </usr/lib/lua/luci/dispatcher.lua:477>
```
OpenWrt 19.07 或更新版本会报此错误，原因是缺少 `luci-compat` 包，使用如下命令安装即可

```sh
opkg install luci-compat
```

- **路由器无法连接 raw.githubusercontent.com**

openwrt 下载 `china_ip_list.txt` 时出现如下错误

```
wget https://raw.githubusercontent.com/17mon/china_ip_list/master/china_ip_list.txt -O /tmp/china_ip_list.txt && mv /tmp/china_ip_list.txt /etc/chinadns_chnroute.txt

--2021-05-16 19:50:02--  https://raw.githubusercontent.com/17mon/china_ip_list/master/china_ip_list.txt
Resolving raw.githubusercontent.com... 0.0.0.0, ::
Connecting to raw.githubusercontent.com|0.0.0.0|:443... failed: Connection refused.
Connecting to raw.githubusercontent.com|::|:443... failed: Connection refused.
```

明显 raw.githubusercontent.com 的 IP 地址被错误地解析到了 0.0.0.0，说明被 GFW DNS 污染了。执行 nslookup 进行验证：

```
root@OpenWrt:~# nslookup raw.githubusercontent.com
Server:         192.168.3.1
Address:        192.168.3.1#53

Name:      raw.githubusercontent.com
Address 1: 0.0.0.0
Address 2: ::
```

发现 openwrt 使用的 DNS 是上游分配的 DNS，按 [Step 4 - 让路由器自身也翻墙](#step-4可选---让路由器自身也翻墙)处理即可。

**iptables v1.8.7 (legacy): unknown option "--to-ports**

如果 shadowsocks 一直无法启动，使用 `/etc/init.d/shadowsocks start` 报如上错误，是因为 OpenWrt 22 开始改用 nftables 来替代 iptables，所以要额外安装：

```
opkg install iptables-legacy iptables-mod-nat-extra ipset iptables-nft
```
