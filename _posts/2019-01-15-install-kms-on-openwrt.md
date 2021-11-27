---
layout: post
title: 搭建 OpenWrt KMS 服务，激活 Office 2016
categories: [Tutorial, OpenWrt]
tags: [OpenWrt]
seo:
  date_modified: 2020-03-02 22:56:24 +0800
---

本文介绍了如何在 OpenWrt 18.06.1 下安装 KMS 服务并激活你的 Windows Office 2016 VOL 版。

## 安装 openwrt-vlmcsd

分别前往项目主页安装

- [openwrt-vlmcsd](https://github.com/cokebar/openwrt-vlmcsd/tree/gh-pages)
- [luci-app-vlmcsd](https://github.com/cokebar/openwrt-vlmcsd/tree/gh-pageshttps://github.com/cokebar/luci-app-vlmcsd/releases)

启动 openwrt-vlmcsd 服务

在 openwrt-vlmcsd 的项界面勾选 `Auto activate` 和`Enable`，然后点击 `Save & Apply`

![启动vlmcsd](/assets/img/post/2019/vlmcsd-config.png)

## 激活 Office 2016

以管理员权限打开 cmd，运行以下命令

```sh
## 32位 Office
cd C:\Program Files (x86)\Microsoft Office\Office16 

## 64位 Office
cd C:\Program Files\Microsoft Office\Office16

cscript ospp.vbs /sethst:192.168.1.1
```

其中 192.168.1.1 替换为你的 OpenWrt 路由器 LAN IP
