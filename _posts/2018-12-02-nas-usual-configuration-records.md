---
title: NAS 相关问题记录
date: '2018-12-02'
categories: [Tutorial, NAS]
tags: [NAS]
seo:
  date_modified: 2020-03-02 22:56:24 +0800
---

## 一些常用命令

- 查找最新修改的文件 (5 分钟内)

```sh
find ./ -mmin -5 -type f
```

- 查看文件夹的大小

```sh
du -h --max-depth=0 /foo/bar
```

## 将频繁写入的日志文件挂载到 /dev/null

```sh
#!/bin/sh
#mount -o bind /dev/null /var/log/scemd.log
mount -o bind /dev/null /var/log/messages
mount -o remount,noatime,commit=600 /
```
## 将 Emby 日志转移至虚拟内存磁盘

Emby 会不间断地写入日志到磁盘，因此可通过软连接的方法将其日志路径改为虚拟内存磁盘 **/dev/shm**

首先在 DiskStation 界面中停用 Emby 服务器，然后 SSH 执行下面的命令

```sh
sudo -i
mount --bind /dev/shm /volume1/@appstore/EmbyServer/var/logs
```

最后，重新启用 Emby 服务器即可

## 关闭 syslog-ng

```sh
sudo -i
vim /etc/init/syslog-ng.conf
```
注释掉启动代码
