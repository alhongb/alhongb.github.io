---
layout: post
title: 使用 Stubby 实现 DNS Over TLS
categories: [Tutorial, OpenWrt]
---

## 介绍

Stubby 是一款跨平台的 DNS Over TLS（DOT）本地代理，本文介绍了 OpenWrt 平台上安装配置 Stubby 并对接现有的 DNS 基础设施 ，使得明文 DNS 请求发送给 OpenWrt（Dnsmasq）后，再由 Stubby 加密发送给支持 DOT 的 DNS 服务器。

## 安装&配置 Stubby

使用 `opkg install stubby` 命令或在 OpenWrt  Web 界面搜索 `stubby` 安装。

Stubby 暂时还没有 LuCi 界面，uci 格式配置文件 位于 `/etc/config/stubby`（根据[文档](https://github.com/openwrt/packages/blob/master/net/stubby/files/README.md#configuration)，当选项 `manual` 配置为 '1' 时忽略 uci 配置文件，使用 `/etc/stubby/stubby.yml` ）。

默认配置下 Stubby 监听 `127.0.0.1@5453` 和 `0::1@5453` 以及使用 Cloudflare 的 DOT 服务器作为上游 DNS，这是我们最关心的两个配置。根据使用目的不同，配置方案分为：

- 场景一方案 - 修改上游 DNS 服务为国内 DOT 服务器（例如例子中的阿里和腾讯），成为「国内安全 DNS」：

```
config stubby 'global'
       option manual '0'
       option trigger 'wan'
       # option triggerdelay '2'
       list dns_transport 'GETDNS_TRANSPORT_TLS'
       option tls_authentication '1'
       option tls_query_padding_blocksize '128'
       # option tls_connection_retries '2'
       # option tls_backoff_time '3600'
       # option timeout '5000'
       # option dnssec_return_status '0'
       option appdata_dir '/var/lib/stubby'
       # option trust_anchors_backoff_time 2500
       # option dnssec_trust_anchors '/var/lib/stubby/getdns-root.key'
       option edns_client_subnet_private '1'
       option idle_timeout '10000'
       option round_robin_upstreams '1'
       list listen_address '127.0.0.1@5453'
       list listen_address '0::1@5453'
       # option log_level '7'
       # option command_line_arguments ''
       # option tls_cipher_list 'EECDH+AESGCM:EECDH+CHACHA20'
       # option tls_ciphersuites 'TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256'
       # option tls_min_version '1.2'
       # option tls_max_version '1.3'

# Upstream resolvers are specified using 'resolver' sections.
config resolver
       option address '223.5.5.5'
       option tls_auth_name 'dns.alidns.com'

config resolver
       option address '223.6.6.6'
       option tls_auth_name 'dns.alidns.com'
	   
config resolver
       option address '162.14.21.178'
       option tls_auth_name 'dot.pub'
```

- 场景二方案 - 保持默认配置（Cloudflare）不变，作为「国外安全 DNS」使用。

显然，场景二的用法在国内大概率会因为 GFW 的作用（DNS 污染）而异常。本文后续基于场景一设置。

关于 Stubby 配置项：

- 配置多个上游 DNS resolver 时，Stubby 将按序使用这些 DNS，解析失败时根据[重试要求](https://github.com/openwrt/packages/blob/master/net/stubby/files/README.md#option-tls_connection_retries)切换至后继的 resolver。
- `address IP` 指示服务器 IP，一方面用于发起 TLS 连接，另一方面当 DOT 连接失败时将回退到传统明文 DNS（当然，默认是禁止回退的）
- 仅当 [tls_authentication](https://github.com/openwrt/packages/blob/master/net/stubby/files/README.md#option-tls_authentication) 选项配置为 'GETDNS_AUTHENTICATION_NONE'(0) 才允许回退明文 DNS，该选项默认是 'GETDNS_AUTHENTICATION_REQUIRED'(1) ，即禁止回退。

## 启动 Stubby

更新 Stubby 配置文件后要在 LuCi - System - Startup 界面点击重启 Stubby 即可。

如果需要手动启动 Stubby，通过 ps 命令可以观察到 Stubby 实际启动命令是 `/usr/sbin/stubby -C /var/etc/stubby/stubby.yml`

其中 `stubby.yml` 的内容如下：

```yaml
tls_authentication: GETDNS_AUTHENTICATION_REQUIRED
tls_query_padding_blocksize: 128
edns_client_subnet_private: 1
idle_timeout: 10000
listen_addresses:
  - 127.0.0.1@5453
  - 0::1@5453
dns_transport_list:
  - GETDNS_TRANSPORT_TLS
upstream_recursive_servers:
  - address_data: 223.5.5.5
    tls_auth_name: "dns.alidns.com"
  - address_data: 223.6.6.6
    tls_auth_name: "dns.alidns.com"
  - address_data: 162.14.21.178
    tls_auth_name: "dot.pub"
```

可见 Stubby 其实是将 uci 格式转换为原生配置格式后再用来启动的。

因此可以采用下面的命令来手动启动 Stubby 进程，来实现多 Stubby 运行实例（注意：不同进程配置监听不同的端口）：

```
/usr/sbin/stubby -C stubby.yml
```

## 对接下游 DNS

Stubby 启用后只是在本地启用了一个 DNS 代理，要真正利用起来还需要根据个人网络环境不同，对接相应的下游 DNS 服务器。

### 对接 Dnsmasq

进入 Luci - Network - DHCP and DNS，将 DNS forwardings 配置为 `127.0.0.1#5453`，打开 Ignore resolve file 选项。这是最简单的方式，Stubby 全局接管 DNS 请求，适用于只需要 DOT 功能无需国际上网的用户。

### 对接 ChinaDNS（不可行）

因为 ChinaDNS 无法显式指定上游 DNS 是国内还是国外（可信），而是自动根据 DNS 的 IP 地址是否是国内来判断「国内 DNS」和「国外 DNS」，想将 127.0.0.1:5453 添加到 ChinaDNS Upstream Servers 作为国内 DNS 的意图实测将会失效（会被误判成国外 DNS，导致解析结果非预期）。具体原因细节参见博主的另外一篇博客[《OpenWrt Shadowsocks 安装&配置指南》- 配置 ChinaDNS](https://linhongbo.com/posts/shadowsocks-on-openwrt/#%E9%85%8D%E7%BD%AE-chinadns) 章节末的对 ChinaDNS 上游 DNS 行为的解释。

### 对接 ChinaDNS-NG

由于 ChinaDNS 的缺陷 ，因此升级为 ChinaDNS-NG 进行对接。方法很简单，将其 China DNS Servers 选项配置为 Stubby 的 `127.0.0.1#5453` 即可。


## 测试

进入 [Cloudflare - Browsing Experience Security Check](https://www.cloudflare.com/ssl/encrypted-sni/) 执行检测，观察第一个检查项 `Secure DNS` 是否 OK。

注：可能需要临时修改 Dnsmasq 的上游 DNS 为 Stubby 来通过测试。
