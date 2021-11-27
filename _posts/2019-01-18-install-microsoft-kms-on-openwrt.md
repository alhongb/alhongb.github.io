---
layout: post
title: ' KMS on OpenWrt router for  activating Windows Office'
categories: [Tutorial, OpenWrt]
tags: [OpenWrt]
lang: en
seo:
  date_modified: 2020-04-05 08:33:44 +0800
---

In this tutorial, I will introduce how to set up a KMS (Key Management Service) on OpenWrt 18.06.01 and activate your Windows Office 2016 VOL (Volume License) editions automatically.

## Install and run openwrt-vlmcsd on your OpenWrt

The First step to activate Window Office is to install **openwrt-vlmcsd** software on your OpenWrt router. openwrt-vlmcsd is an OpenWrt package for vlmcsd which emulates a KMS to supports Microsoft products activation.

First, to install openwrt-vlmcsd, go to the projects' homepage and download the pre-compiled packages. Note that the downloaded `ipk` should corresponds to your hardware platform.

- [openwrt-vlmcsd](https://github.com/cokebar/openwrt-vlmcsd/tree/gh-pages)
- [luci-app-vlmcsd](https://github.com/cokebar/openwrt-vlmcsd/tree/gh-pageshttps://github.com/cokebar/luci-app-vlmcsd/releases)

```sh
## You need download the package corresponds to your platform, in my case, is x86_64
wget https://github.com/cokebar/openwrt-vlmcsd/blob/gh-pages/vlmcsd_svn1112-1_x86_64.ipk
opkg install vlmcsd_svn1112-1_x86_64.ipk

wget  https://github.com/cokebar/luci-app-vlmcsd/releases/download/v1.0.2-1/luci-app-vlmcsd_1.0.2-1_all.ipk
opkg install luci-app-vlmcsd_1.0.2-1_all.ipk
```

And finally, enable openwrt-vlmcsd service:

Check "`Auto activate`" and "`Enable`" options in the LuCI app then click "`Save & Apply`".

![openwrt-vlmcsd config](/assets/img/post/2019/vlmcsd-config.png)

## Activate your Office 2016

Run `cmd` as administrator on your Windows, and then execute following commands. Note that you should replace the IP (in this case is `192.168.0.1`)  with your own OpenWrt LAN IP.

```sh
## for 32bits Office:
cd C:\Program Files (x86)\Microsoft Office\Office16 

## for 64bits Office: 
cd C:\Program Files\Microsoft Office\Office16

cscript ospp.vbs /sethst:192.168.0.1
```
