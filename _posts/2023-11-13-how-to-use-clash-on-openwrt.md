---
layout: post
title: OpenWrt 下安装和使用 Clash (ShellCrash)
categories: [Tutorial, Clash]
tags: [OpenWrt, Clash]
---

## 简介

Clash 是非常流行的流量代理软件，特别是很多机场将其作为首选的客户端。而其中的 [ShellCrash](https://github.com/juewuy/ShellCrash/tree/master)（这里没有拼写错误，并不是 ShellClash） 则是包括 OpenWrt 在内的一众 Linux 系统最佳 Clash 客户端之一，其最大亮点是安装配置非常便捷、Linux 友好、支持 Web 图形界面。下文就介绍如何在 OpenWrt（本文使用 23.5.0）下快速安装配置 ShellCrash。当然，其他基于 Linux 系统理论上也是适用的。 

## 安装 ShellCrash

SSH 进入系统后，按照 ShellCrash 项目主页[在线安装](https://github.com/juewuy/ShellCrash/blob/master/README_CN.md#%E5%9C%A8%E7%BA%BF%E5%AE%89%E8%A3%85)的命令进行安装，这里推荐使用作者私人源进行安装，因为此时的路由器通常访问 Github 速度慢甚至无法访问：

```
export url='https://gh.jwsc.eu.org/master' && sh -c "$(curl -kfsSl $url/install.sh)" && source /etc/profile &> /dev/null
```

根据提示进行安装即可。这里 ShellCrash 对于安装目录空间不足 10MB 的设备会提示是否开启小闪存模式，使得核心及数据库文件会被下载到内存中，但实际上博主实测完整安装 ShellCrash 只需不到 5MB 空间。

## 配置和启动 ShellCrash

安装完 ShellCrash 后就可以直接执行 `clash` 命令进行运行前的最后配置，我们根据提示进入`导入Clash配置文件链接`功能，将订阅链接粘贴导入进去，并根据提示重启 Clash 服务。到这里就基本大功告成了，接下来我们需要一个管理订阅规则的控制面板。

有多种控制面板可供选择，你可以通过访问在线地址来直接使用控制面板：

|面板类型|在线地址|
|-----|-----|
|Clash 官方面板|http://clash.razord.top|
|Razord-meta 面板|http://clash.metacubex.one|
|yacd 面板|http://yacd.haishan.me|
|Yacd-meta 面板|http://yacd.metacubex.one|
|metacubexd 面板|http://d.metacubex.one|

使用在线控制面板时需要配置主机的 host 和端口（ShellCrash 启动 Clash 时提示），但如果你的 host 非 https，那么你应该确保访问在线面板的 http 版本，否则将被浏览器拒绝连接。

当然，也可以直接将面板安装到 OpenWrt 上：

直接通过 ShellCrash 的`更新/卸载`选项进入`安装本地Dashboard面板`功能即可执行安装，例如：

```
-----------------------------------------------
安装本地版dashboard管理面板
打开管理面板的速度更快且更稳定
-----------------------------------------------
请选择面板安装类型：
-----------------------------------------------
 1 安装官方面板(约500kb)
 2 安装Meta面板(约800kb)
 3 安装Yacd面板(约1.1mb)
 4 安装Yacd-Meta魔改面板(约1.5mb)
 5 安装MetaXD面板(约1.5mb)
 6 卸载本地面板
 0 返回上级菜单
```

博主选择其中的 [yacd 面板](https://github.com/haishanh/yacd):

![yacd 面板](https://user-images.githubusercontent.com/1166872/47954055-97e6cb80-dfc0-11e8-991f-230fd40481e5.png)

安装完成后，根据提示访问 `http://192.168.0.1:9999/ui/` 即可通过面板来对订阅的规则进行配置。

## 其他配置

### 定时更新订阅

通过`设置定时任务`可以定时更新订阅、重启 Clash 服务。

### 让路由器本机也通过 Clash 流量代理

默认情况下，ShellCrash 不会让路由器本机也代理流量，如果要让路由的流量也可以被代理，可以通过 `clash功能设置`-`设置本机代理服务`来开启这一功能。


