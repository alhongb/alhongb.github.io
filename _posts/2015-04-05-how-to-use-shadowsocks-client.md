---
layout: post
title: Shadowsocks 客户端使用教程
categories: [Tutorial, Shadowsocks]
tags: [Shadowsocks]
seo:
  date_modified: 2020-03-02 22:56:24 +0800
---

## 简介

想用 Google 搜资料却碰到404错误？想用 Facebook 交流却连账号都注册不了？没问题，本文首先简要介绍了 Shadowsocks 的工作原理，然后详细说明了各个平台下使用 Shadowsocks 客户端的方法，按照本文的方法你就可以正常访问 Google 了。

## Shadowsocks 工作原理

![Shadowsocks 工作原理图](/assets/img/post/2015/how-shadowsocks-work.gif)

如图所示，由于防火墙（GFW）作为出口网关级的存在，中国境内发起的所有境外访问均会可被随意阻断，通常这种阻断组合利用了 IP 地址拦截、 DNS 污染、流量识别等多项技术。但目前为止，防火墙的封禁还是基于黑名单机制，未在黑名单的境外服务器依然能够正常访问。基于此，借助 Shadowsocks 客户端和位于境外的 Shadowsocks 代理服务器就能够间接地绕过防火墙，同时客户端与代理服务器间的通信数据采取了加密保护，使得防火墙无法正常识别流量。

## 各平台客户端使用教程
### Android

1. 安装 [Android 客户端](https://github.com/shadowsocks/shadowsocks-android/releases)

2. 配置好服务器端信息（可以通过剪贴板导入、扫描二维码、手动配置等方式）

3. 打开客户端，点击启动按钮，不出意外，你已经可以访问 Google 了。

**提示**：在客户端路由选项里选择绕过局域网及中国大陆地址可优化外网冲浪体验；如果你添加了多个服务器，可以通过影梭的菜单栏来切换选择；你还可以通过分应用代理进一步优化上网体验。

### iOS

推荐使用 [Shadowrocket](https://itunes.apple.com/us/app/shadowrocket/id932747118?mt=8)

由于 Apple Store 国区已全部下架翻墙 App，只好使用企业账户应用分发渠道下载盗版 Shadowrocket，具体方法如下：

1. 进入 [https://ios.ss-ssr.com](https://ios.ss-ssr.com)，按页面提示安装好 Shadowrocket

2. 打开 Shadowrocket，在弹出登录框中用网站提供的 Apple ID 登录 Apple Store（该帐号已购买应用）以激活应用。

**注意：**激活成功后你可在 Apple Store 中注销这个 Apple ID，用自己的 Apple ID 恢复登录; 不要使用三方 Apple ID 登录你的 iCloud，否则存在隐私和安全风险！

### Windows

1. 下载 Windows 版  [Windows 版客户端](https://github.com/shadowsocks/shadowsocks-windows/releases)

2. 解压缩包至你喜欢的目录

3. 启动 Shadowsocks

双击击运行 shadowsocks.exe ，配置好你的远程服务器，选择 PAC模式。不出意外，此时你的 IE 浏览器已经可以正常访问 Google 了。

如果你日常使用 Chrome ，请参考下文 Linux 客户端的配置方法，安装 SwitchyOmega 扩展来优化网上冲浪体验。

### Mac OS

以后再写吧...

### Linux

1. 安装 Shadowsocks 客户端（Python 版）


```sh
## Debian/Ubuntu
sudo apt-get install python-pip
sudo pip install shadowsocks

## CentOS
sudo yum install python-setuptools
sudo easy_install pip
sudo pip install shadowsocks
```

你也可以到 [这里](http://shadowsocks.org/en/download/clients.html) 选择你喜欢的 Shadowsocks 版本

2. 配置 Shadowsocks 客户端和 Chrome

创建一个配置文件 shadowsocks.json，内容如下：

```
{				
	"server":"服务器地址",
	"server_port":服务端端口号,
	"local_address": "127.0.0.1",
	"local_port":1080,
	"password":"密码",
	"method":"加密方式"
}
```

给你的 Chrome 安装 [SwitchyOmega](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif)，并根据需要配置代理。

3. 通过 Shadowsocks 代理上网

启动 Shadowsocks


```sh
sslocal -c /..你的路径../shadowsocks.json
```

SwitchyOmega 选择 自动切换（仅国外代理）或者 Shadowsocks （完全代理）

如图所示

![SwitchyOmega 界面](/assets/img/post/2015/switchyomega.png) 

不出意外，你已经可以访问 Google 了 :)

### OpenWrt 客户端

参考博主新近写的 [OpenWrt Shadowsocks 安装&配置指南]({% post_url 2018-07-21-shadowsocks-on-openwrt %})
