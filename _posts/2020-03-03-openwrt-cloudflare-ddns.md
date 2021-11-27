---
title: OpenWrt Cloudflare DDNS
categories: [Tutorial, OpenWrt]
tags: [DDNS]
layout: post
seo:
  date_modified: 2020-03-24 21:13:55 +0800
---

本文演示了如何在 OpenWrt 上安装和设置 DDNS 软件包，最后接入 Cloudflare 提供的 DDNS 服务。

## 安装 DDNS 软件包

要使用 Cloudflare DDNS，你需要安装 `ddns-scripts` 和 `ddns-scripts_cloudfare.com-v4` 两个软件包：

```sh
opkg install ddns-scripts ddns-scripts_cloudfare.com-v4
```

其中前者是 DDNS 服务主功能脚本，后者是它的 Cloudflare 支持。

## 获取 Cloudflare API Key

DDNS 的原理其实很简单，就是客户端定期通过 DNS 服务商提供的 API 来修改指定域名的 DNS 记录。出于访问控制要求，DNS 服务商一般要求接口调用者提供身份认证凭据，比如下面提到的 API Key。

Cloudflare 提供的 DDNS API 是 RESTful 形式，客户端调用时必须使用 Cloudflare 帐户对应的 API key，即身份凭证。可前往 [My account](https://dash.cloudflare.com/profile/api-tokens) 页面获取你的 API Key。例如

```
ebcdefghijklmnopqrstuvwxyze1234567890
```

## 配置 DDNS

在 `Dynamic DNS` LuCI 界面中，新建一个 DDNS 配置项，内容如下：

- 勾选 `Enabled`
- `Lookup HostName`: 要执行 DDNS 的完全限定域名（FQDN），例如 `subdomain.example.com`
- `IP address version`: 勾选 `IPv4-Address`
- `DDNS Service provider`: 选择 `cloudfare.com-v4`
- `Domain`: 按“主机名@域名”的格式填写，例如 `subdomain@example.com`
- `Password/密码`：填写先前获取的 Cloudflare API key
- `Optional Parameter`: 填写 `"proxied":false`。由于 Cloudflare API 会默认开启 `proxied` 选项（就是 Cloudflare DNS 面板点亮域名旁边黄色的云朵，表示该域名已被 Cloudflare 执行 CDN 代理），而我们家用 DDNS 一般只是为了获取 IP 地址，被 Cloudflare 代理以后反而会获取不到真实的 IP 地址，所以这里一般要关闭这个选项，转变为 `DNS only` 模式 (即让 Cloudflare 面板黄色的云朵变灰)，否则你的域名会自动开启 `proxied` 功能

以下选项表示要求使用 HTTPS 安全通道访问 Cloudflare，出于安全考量，建议打开

- 勾选 `Use HTTP Secure`
- `Path to CA-Certificate`: 填写 `/etc/ssl/certs` 注：OpenWrt 默认未预置 CA 根证书，使用如下命令安装

```sh
opkg install ca-certificates
```

最后，点击 `Save & Apply` 观察 DDNS 是否生效即可。
