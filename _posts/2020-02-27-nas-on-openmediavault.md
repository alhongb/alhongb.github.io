---
title: 搭建 openmediavault NAS
categories: [Tutorial, NAS]
tags: [openmediavault, NAS]
layout: post
seo:
  date_modified: 2020-04-28 22:53:42 +0800
---

本文详细记录了在 openmediavault 上搭建私人 NAS 的过程，包括：安装配置 openmediavault、Docker；部署 Transmission BT 工具、Nextcloud 网盘等容器；配置 HTTP/HTTPS 反向代理和 Let's Encrypt 证书，最终实现个人 NAS 的搭建。

## 为什么选择 openmediavault？

简要对比部分主流 NAS 操作系统，博主认为 openmediavault 更优

- Synology DSM

    适合无技术背景或希望开箱即用的用户。博主不选择的原因：

    1. 相对臃肿，不够简洁
    2. 每块磁盘上都会安装操作系统以及后续安装的软件及其数据，导致用户存储和操作系统耦合，频繁的读写影响每块硬盘的休眠。
    3. DIY 设备安装运行涉及版权，无法稳定升级版本

- FreeNAS

    对硬件要求比较高，尤其是内存最低要求 8G，其未来的目标用户应该主要是企业，不选。

- openmediavault

    基于 Debian Linux，开源免费。openmediavault 的目标就是面向家庭和小型办公环境，是对熟悉 Linux 又追求最小化安装的人的首选。

## 硬件选购建议

博主用过的 DIY NAS 硬件有小马 V5（已退役）和蜗牛星际，这里提一些硬件选购建议，供读者参考：

- 专用设备

NAS 的核心功能应当是可靠的数据存储，长期稳定运行是一大要素，因此不建议使用虚拟化技术（一设备多用途）、树莓派等非专用设备来构建你的 NAS。

- 低功耗

低功耗不仅意味着绿色清洁，还带来更好的散热性能，这些都是 NAS 设备长期运行的基础

- 盘位至少 3，最优 4

个人认为兼顾数据安全和丰富应用的硬盘布置策略是：占用 2 盘位的 `RAID 1`（mirror）+ 1 盘位单盘（或 2 盘位 `RAID 0`），前者用于存储个人数据或稀缺资源，后者用于 BT、电影分享等数据容易重新下载的场景。因此满足此策略的 4 盘位硬件就足矣，至于超过 4 盘位的，个人觉得不必要。

另外，所有的 NAS 系统都有物理机裸装和运行在 ESXi 等虚拟化平台上两种区分，但考虑到虚拟机对 S.M.A.R.T、磁盘休眠等需要硬件直通的特性支持不好，而这些功能是 NAS 长期稳定、低功耗运行的核心，因此强烈建议不要使用虚拟化安装 NAS。

更新：2 盘位实际上也可以采用 rsync 目录同步软件来实现双盘备份，不失为更经济实用的方案。

## 安装 openmediavault

openmediavault 基于 Debian，因此安装过程与绝大部分 Linux 发行版没什么两样：先在[官网](https://www.openmediavault.org/?page_id=77)下载 ISO 文件，解压到 U 做成启动盘，最后引导设备启动到安装程序完成安装。几个注意事项：

- 在安装界面执行磁盘重新分区时，可能会报无法安装系统文件的错误，忽略错误重启一次即可
- 网站提供的 ISO 镜像最新版本是 5.0.5，但安装完成后会自动升级到最新版本
- 安装过程确保联网更新，避免网卡驱动安装异常（重启动后无法获取 IP 地址）等问题。更新源选择国内的节点，如清华，否则速度极慢
- 安装过程语言选项可选择英文或中文，影响 Debian 和 openmediavault 的语言。建议选英文，因为 openmediavault 的时区、语言安装完成后很容易通过其界面更改

## 设置共享文件夹

### 创建共享文件夹

openmediavault 成功运行后，就可以用其 Web 图形界面组建 RAID 和创建可被外部访问的`共享文件夹`了，基本流程是：

```
清除磁盘（可选）- 组建 RAID（可选）- 创建文件系统 - 挂载文件系统 - 添加共享文件夹 - 打开文件共享服务（可选）
```

注意：默认的 `admin` 管理员用户不能用于访问共享文件夹，需要新建一个普通用户；所有共享文件访问都应当通过 openmediavault `共享文件夹`机制：主机外使用各种基于网络的文件共享服务，主机内使用 `/srv/dev-disk-by-label-xxxx/yoursharedfolder` 路径来访问共享文件夹（请注意，在最新的 openmediavault 上，`/sharedfolders/` 已废止使用）。

### 多用户模式（可选）

如果你的 NAS 是个人专用，不考虑家庭共享，可以忽略本小节。但是如果你的 NAS 要满足一大家庭或为此做打算，可以如下两种模式中选择一种来设计多用户和共享文件夹，使得不同用户的数据互相隔离。

#### 模式一：使用主目录

1. 创建用户

    根据实际需求创建，你、你的家人...

2. 开启`使用主目录`选项

    前往 `访问权限管理` - `用户` - `设置` 启用主目录功能，让 openmediavault 为每个用户分配专属家目录。这里 home 目录需要绑定到一个共享文件夹，可以事先创建名为 home 的共享文件夹，该目录属主、用户组应为 root:users，其中 user 组仅有 x 权限，其他用户无权限。

3. 使 Samba 访问时只显示该用户的主目录，不显示多余的 `homes` 目录

    前往 `服务` - `SMB/CIFS` - `主目录` - 选中 `启用用户主目录`，取消选中`设为可浏览`

按照上面方法，每个用户都将拥有独立的数据目录，并且不同用户间互相隔离仅对属主开放权限。

至于整个家庭范围内共享的数据，可以额外按需创建共享目录。例如：创建一个名为“家庭共享”的共享目录，将所有家庭范围分享的数据放置在此目录下：由于 openmediavault 新建用户默认都属于 `user` 组，对于这类共享目录，整个家庭都将可以访问。

#### 模式二：为每个用户创建独立的共享文件夹

对于每个新建的用户，单独设立一个共享文件夹，并且对每个共享文件夹执行 ACL 设置隔离规则：

- 属主修改为该用户
- 用户组修改为该用户对应的专用用户组，而非 users 组（这要求你提前建立好用户组）

修改好以后，不同用户间就没有权限访问对方的共享目录了，但是在 Samba 中，无权限者还是会看到这些共享目录，要对无权限者屏蔽这些共享目录，可在 SMB/CIFS 中对这些共享目录设置一条 `extra options`，内容为：

```
access based share enum = yes
```

同时在 Shared Folders - Privileges 选项中显式拒绝其他用户的访问权限。（正如其说明，Privileges 只会影响共享服务的权限配置，而不会反映到文件系统）

## 安装 Docker 环境

### 安装 Docker

为了运行 Nextcloud（云盘）、Transmission（BT 下载）等软件，需要安装和配置 Docker 运行环境。

首先按其官网的方法安装三方扩展插件库 [omv-extras](http://omv-extras.org/) ：

```sh
sudo wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash
```

安装完毕后，直接在 openmediavault 新增的 `omv-extras` 选项中点击按钮安装 Docker 即可

### 启用 user namespace

默认情况，docker 容器内的 UID 0 用户会映射到主机的 root 用户，因此默认情况下绝大部分容器进程就是以本机 root 用户运行，非常不雅。虽然可以通过指定容器运行的用户来规避，但是实际运用会发现很多镜像会报权限无法正常运行，好在新版本的 docker 已经支持 Linux user namespace 安全能力，可使得容器内的「root」直接映射为主机上的普通用户。

**注意**：务必在创建第一个镜像之前就执行此步骤，否则在启用 user namespace 之前已经创建的容器、volume 等都会在启用后将「冻结」（就是完全隐藏，也无法使用），直到 docker 退出 user namespace 模式。

要启用 docker 的 user namespace 隔离功能，编辑 `/etc/daemon.json` 如下

```json
{
  "userns-remap": "default"
}
```

其中 `default` 表示使用 docker 默认创建的用户、用户组来执行映射，也可以自行指定要映射的用户。详细配置方法参考[官网](https://docs.docker.com/engine/security/userns-remap/)。

开启 userns-remap 后，使用 systemctl restart docker 使之生效，并可以使用以下命令查看是否生效：

```sh
$ grep dockremap /etc/subuid
dockremap:231072:65536
$ grep dockremap /etc/subgid
dockremap:231072:65536
```

上面的 subuid/subgid 是 Linux 的从属（subordinate）uid/gid 配置文件， 其中 `dockremap:231072:65536` 被分配了从 231072 开始，  231072 + 65536 结束的 uid/gid 范围。这样，docker 容器内的 uid/gid `0` 就会映射到主机上的 uid/gid 231072，而 uid/gid `1` 则映射为主机上的 uid/gid 231072，以此类推。显然 uid/gid 231072 在主机上是毫无权限的，甚至并非是一个真实存在的用户或用户组。

此后运行容器，通过 `ps` 命令可以看到容器进程的 uid 不再是 root，而是 231072

当然，有些场景容器需要高权限运行，比如下面提到的 Portainer 这时，就可以用 `--userns=host` 参数来豁免 user namespace 隔离。

### 安装 Portainer

Portainer 是一个轻量级的 Docker 图形化管理工具，可以用来管理宿主机和 Docker Swarm 集群。Portainer 本身也是以容器运行，因此安装过程就是创建和运行一个容器。

omv-extras 也提供了 Portainer 的安装界面，直接用它来安装即可。（**注意**：若启用了 docker namespace，需手动安装，方法见下）

安装完成后访问 *openmediavualt_host_ip*:9000 即可访问其 Web 界面，首次登录需要创建一个管理员用户，然后会让你选择要连接的 Docker 环境，这里我们选择 `Local`，即此前安装的 Docker。

可选：手动安装 Portainer（假设 docker 已经启用了 docker user namespace）：

进入 https://www.portainer.io/install 选择 CE 版本，按官方指导进行安装（添加  `--userns=host` 参数以适应启用了 docker user namespace），例如：

```sh
docker volume create portainer_data
docker run -d -p 9000:9000 -p 8000:8000 -p 9443:9443 --userns=host --name portainer \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:2.11.0
```

要升级 Portainer：

```sh
docker stop portainer
docker rm portainer
```

然后重新执行上面的 docker run 命令即可

## 部署 NAS 常用应用容器

### docker container 命令基础

- Docker 默认以 root 运行容器进程，除非 `docker container` 命令指定了 `--user` 参数
- `docker container run` 命令是创建并运行一个容器，`docker container create` 则仅创建容器而不运行。要运行已有的容器，使用 `docker container start` 命令
- `--rm` 参数是容器运行结束后自动删除该容器文件，由于默认情况下 Docker 每次启动容器都会新创建一个对应的容器文件， 对于仅一次性运行容器的场景，`--rm` 参数就会很有用
- `-v` 参数说明：

Docker 支持三种文件系统实现：`volume`，`bind mount` 和 `tmpfs mount`，它们分别用不同的方式将容器内的文件/目录和宿主的文件/目录关联，使容器能够访问宿主的文件系统/内存文件系统。 其中 `volume` 是 Docker 自行管理的文件/目录，也是最易使用的：通过 `-v` 参数指定一个唯一的 volume name 以及要关联到的容器内文件/目录即可。`volume` 由 Docker 管理，使用时可以是已经存在或未存在的任一个，当不存在时 Docker 会负责创建；`bind mount` 是将容器内的目录/文件绑定到宿主上的已有的目录或文件，用法是和 `volume` 类似，但 `-v` 参数指定的一定是宿主机上某个相对路径或绝对路径。

![docker volumes](https://docs.docker.com/storage/images/types-of-mounts-volume.png "三种 Docker 文件系统关系示意图")

- `-e` 参数，增加一个供容器使用的环境变量

- `PUID/PGID`，是 [LinuxServer.io](https://www.linuxserver.io/) 组织提供的镜像特有的实用功能，作为环境变量指定，以指定容器进程运行所用的 UID/GID

### 创建容器专用用户（可选）

出于安全考虑，创建一个专用的低权限用户来运行各项容器，用以实践权限最小化原则。

首先直接在 openmediavault 界面中创建一个名为 application 的用户（默认的用户组为 users）。

那么，未后续如何使用这个新用户来应用权限最小化安全实践？概括而言就是两个方面操作：

1. 将用户 ID 后续作为所有容器的启动参数，使得容器进程以 application 用户身份和和组运行；
2. 对于需要限制容器访问的目录，通过 ACL 限制 application 用户对该目录的访问。

>你可能有疑问，如果容器服务以 application 的 UID 和 GID 运行，在共享目录创建的文件属主将是 application，那其他用户能也访问吗的？
>
>答案是肯定的。这是因为 openmediavault 共享目录的用户组都是 users，并默认有 setgid 标志位，这使得容器服务在其下创建的子目录和文件将都与共享目录一致，即 users 组。

### 部署 Transmission 容器

Transmission 用来下载 BT（支持磁力链接）非常不错，支持 Web 界面和包含认证的 RPC 控制，我们选择 linuxserver 提供的镜像，[Docker Hub - linuxserver/transmission](https://hub.docker.com/r/linuxserver/transmission/)

容器安装是用 Portainer，具体操作步骤是：

1. 进入 `Volumes` - `Add volume` 页面创建 Transmission 容器配置数据存储专用 `volume`

    - 填写 `Name`，例如 `transmission_config`
    - 点击 `Create the volume` 创建该 `volume`

2. 进入 `Containers` - `Add container` 页面配置容器参数

    - 填写任意的 `Name`，填写 `Image` 为 `linuxserver/transmission`
    - `Manual network port publishing` 中点击 `publish a new network port`，按 linuxserver/transmission 在 Docker Hub 页面的要求依次添加端口映射
    - `Volumes` - `Volume mapping` 选项中将专用于存储配置数据的 `Volume` 即 `transmission_config`，绑定到 `/config` 和 `/watch`；`Bind` 形式绑定 `/downloads` 到你指定的 NAS 共享目录。
    - `Env` - `Environment variables` 中添加 `PUID`、`PGID` 两个环境变量，如果只想方便可分别填写 `1000` 和 `100` 即你的 openmediavault 使用用户和 `users` 组，如果想最小化运行权限，可填写[创建容器专用用户](#创建容器专用用户可选)一节创建的 application 用户的 UID 和 GID，同时将第 1 步绑定 downloads 的共享文件夹目录属主修改为 `application`（用户组保持 users 不变）,开放给容器访问。
    - `Restart policy` 中选择 `Unless stopped`
    - 最后点击 `Deploy the container` 完成容器部署

### 部署 Nextcloud 容器

*建议使用 File Browser 替代*

Nextcloud 作为云盘软件，实际上主要的文件管理功能完全可以使用 openmediavault 提供的 Samba/NFS 替代，但是如果要在 Internet 上分享文件，后者就十分乏力了。

我同样选择 linuxserver 提供的镜像，[Docker Hub - linuxserver/nextcloud](https://hub.docker.com/r/linuxserver/nextcloud)。之所以不选择官方镜像，是因为其不支持设置容器进程的 UID/GID，无法控制容器进程的读写权限。

Nextcloud 容器运行起来后，还要编辑一下它的配置文件，将域名修正为你自己的实际域名，我的例子是 `nextcloud.linhongbo.com`

```sh
vim /var/lib/docker/volumes/nextcloud_config/_data/www/nextcloud/config/config.php

<?php
$CONFIG = array (
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'datadirectory' => '/data',
  'instanceid' => 'oc2sfcyt03u3',
  'passwordsalt' => '1234567890abcdefg...',
  'secret' => '1234567890abcdefg...',
  'trusted_domains' =>
  array (
          0 => 'openmediavault',
          1 => 'nextcloud.linhongbo.com',
  ),
  'dbtype' => 'sqlite3',
  'version' => '18.0.1.3',
  'overwrite.cli.url' => 'https://nextcloud.linhongbo.com:8443',
  'installed' => true,
);
```

### 部署 File Browser 容器

相比 Nextcloud，File Browser 更轻量，博主推荐使用 File Browser，轻量、界面简洁。

File Browser 有两个 docker 镜像可供选择：[官方镜像](https://hub.docker.com/r/filebrowser/filebrowser)，[更好用的镜像](https://hub.docker.com/r/hurlenko/filebrowser)，推荐使用后者。因为官方镜像无法使用普通用户启动（容器绑定 80 端口需要 root），且配置项也比较奇怪。

不想使用 root 用户来运行的注意：Portainer 配置容器参数时，可将 `user` 参数配置成普通用户的 UID 和 GID，形如：1000:100，同时容器分配、绑定的 volume 目录属主也要相应修改，避免权限问题。假设分配给容器的 volume 名为 `file_browser`，则要修改属主为对应普通用户的目录是：

```
drwxr-xr-x 3 hongbo users 4096 11月 29 08:16 /var/lib/docker/volumes/file_browser/_data
```

### 部署 Emby Server 容器

相比 Plex 博主更喜欢 Emby。镜像地址：[Docker Hub - emby/embyserver](https://hub.docker.com/r/emby/embyserver/)

同样的，embyserver 容器的 `UID/GID` 环境变量如果只想方便可分别填写 `1000` 和 `100` 即你的 openmediavault 使用用户和 `users` 组，如果想最小化运行权限，可填写[创建容器专用用户](#创建容器专用用户可选)一节创建的 application 用户和用户组，同时将第 Emby 的 volume 目录属主修改为 `application`（用户组保持 users 不变）,开放给容器访问：
    
```sh
chown application transmission_volume_directory
```

### 使用域名访问容器

默认情况下，我们会用 `IP:端口` 方式访问容器提供的 Web 服务，并且用端口号来区分不同的服务，这在容器数量较多时显得非常不优雅。

有两种解决方案，分别适用不同场景：

1. 如果你有公开域名并希望容器能够被公网访问，可以在域名 DNS 服务提供商中配置新的容器服务域名，并解析到可访问容器的公网 IP（这通常需要宽带运营商提供公网 IP，并且你已[配置好 DDNS](https://linhongbo.com/posts/openwrt-cloudflare-ddns/)）
2. 如果你没有公开域名或仅在局域网访问容器服务，可以配置本地域名。具体方法不一，对于 OpenWrt 来说，以下两种方法均可

- 在 Luci Web 界面 `Network - Hostnames` 中添加域名和对应的 IP 地址

- 在 `/etc/hosts`，中添加域名和对应的 IP 地址，形如：

    ```
    192.168.0.2 portainer.linhongbo.com
    ...
    ```

## 启用 HTTPS

出于安全考虑，一些 Web 服务需要启用 HTTPS，尤其是对公网暴露或涉及账户口令的。这里博主采用 Let's Encrypt 的通配证书方案，并以 openmediavault-webgui 为例启用 HTTPS。

### 申请 Let's Encrypt 通配证书

2018 年 3 月 Let’s Encrypt 终于宣布支持 ACME v2 and Wildcard Certificate，即通配符证书。非常振奋人心，Good Job！申请通配符证书需要在域名的 DNS 上配置 `TXT` 记录，我们先尝试手动模式申请通配符证书，然后运行一个对应于域名 DNS 服务商的 Certbot 容器来自动申请

- 手动申请

安装 Certbot

```sh
sudo apt install certbot
```

接着运行如下命令给你的域名申请统配符证书（将命令中的域名替换成你自己的）

```sh
certbot certonly --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory --manual-public-ip-logging-ok -d '*.linhongbo.com' -d linhongbo.com
```

Cetbot 会先提示提供邮箱，这里如实填写，以接收证书过期提醒邮件；然后要求配置一个指定值的名为 `_acme-challenge` 的 DNS TXT 记录，我们到自己的 DNS 服务商面板上配置即可；最后稍等片刻，先用 [dns.google.com](https://dns.google.com) 查询一下 TXT 记录是否生效，确认生效后回车。待 Let's Encrypt 校验域名所有权后，会签发证书，成功申请的输出如下：

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/linhongbo.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/linhongbo.com/privkey.pem
   Your cert will expire on 2018-06-12. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - If you like Certbot, please consider supporting our work by:
 
 
   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

Cetbot 将证书和私钥归档在 `/etc/letsencrypt/archive/yourdomain.domain/` 下，但我们应通过其额外提供的软连接访问，即

```
/etc/letsencrypt/live/linhongbo.com/fullchain.pem
/etc/letsencrypt/live/linhongbo.com/privkey.pem
```

- 自动申请

自动申请能够自动化配置 DNS TXT 记录，这需要 Certbot 安装对应域名服务商的插件，处于方便考虑，直接采用 Docker 容器运行。

首先，创建一个用于访问 DNS 服务商 API 的接口配置文件，内容根据你实际 DNS 服务商提的而不同，博主例子是 Cloudflare

```sh
vim cloudflare.ini

# Cloudflare API credentials used by Certbot
dns_cloudflare_email = cloudflare@example.com
dns_cloudflare_api_key = 0123456789abcdef0123456789abcdef01234
```

其中 `dns_cloudflare_email` 为 Cloudflare 账号邮箱；`dns_cloudflare_api_key` 前往 [Cloudflare - Profile - API Tokens](https://dash.cloudflare.com/profile/api-tokens) 获取（这里选则创建权限 `Edit zone DNS` 的 Token 即可，而不是 `Global API Key`）。

接着在 Docker Hub 上查找到对应的 [Certbot 镜像](https://hub.docker.com/u/certbot/)，我的例子是 certbot/dns-cloudflare，然后运行如下命令申请证书

```sh
sudo docker run -it --rm --name certbot \
            -v "/etc/letsencrypt:/etc/letsencrypt" \
            -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
            -v "/root/cloudflare.ini:/cloudflare.ini" \
            certbot/dns-cloudflare certonly --preferred-challenges dns --dns-cloudflare --dns-cloudflare-credentials /cloudflare.ini  -d *.linhongbo.com -d linhongbo.com --server https://acme-v02.api.letsencrypt.org/directory
```

其中 `/root/cloudflare.ini` 替换为你自己的文件路径。

Letsencrypt 证书三个月过期，到期 renew 时再执行上述 Docker 命令即可，此时会提示证书可以 renew。

### 为 openmediavault-webgui 启用 HTTPS

openmediavault 界面提供了 HTTPS 配置功能，但密钥管理功能很不友好，建议直接采取以下手动方法来启用 HTTPS

创建额外的 openmediavault-webgui 配置文件：

```sh
cd /etc/nginx/openmediavault-webgui.d
touch custom.conf
vim custom.conf
```

添加如下配置：

```
listen [::]:443 default_server ipv6only=off;
ssl_certificate /etc/letsencrypt/live/linhongbo.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/linhongbo.com/privkey.pem;
```

最后使用 nginx -s reload 加载配置即可

## Nginx 反向代理

反向代理可以使局域网内的 Web 服务直接通过标准的 HTTP/HTTPS 域名、端口（80/443）访问。openmediavault 预置安装了 Nginx（被其管理界面使用），我们直接复用这个 Nginx 为本机上的各个容器或其他主机上的服务提供反向代理。

有两种反向代理配置模式：一种是每个为目标 Web 服务创建一个代理 Virtual Host，另一种则是直接在主 Host 下分配 `location` 目录。前者需要为每个代理 Host 增加 DNS 记录，配置和维护起来稍显麻烦，好处是对浏览器比较友好；后者则只需维护一个主 Host 的 DNS，配置起来也相对简洁容易。

### location 目录反向代理

创建并编辑 `/etc/nginx/openmediavault-webgui.d/proxy.conf`，按如下可用的 filebrowser、Portainer、Emby、transmission 的反向代理配置：

```
location /filebrowser {
    # prevents 502 bad gateway error
    proxy_buffers 8 32k;
    proxy_buffer_size 64k;

    client_max_body_size 75M;

    # redirect all HTTP traffic to localhost:9001;
    proxy_pass http://localhost:9001;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #proxy_set_header X-NginX-Proxy true;

    # enables WS support
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 999999999;
}

location /portainer/ {
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header Connection "";
    proxy_pass http://localhost:9000/;
}
location /portainer/api/websocket/ {
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_http_version 1.1;
    proxy_pass http://localhost:9000/api/websocket/;
}

location /emby/ {
    # prevents 502 bad gateway error
    proxy_buffers 8 32k;
    proxy_buffer_size 64k;

    client_max_body_size 75M;

    # redirect all HTTP traffic to localhost:9001;
    proxy_pass http://localhost:9003/;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #proxy_set_header X-NginX-Proxy true;

    # enables WS support
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 999999999;
}

location /transmission {
    # prevents 502 bad gateway error
    proxy_buffers 8 32k;
    proxy_buffer_size 64k;

    client_max_body_size 75M;

    # redirect all HTTP traffic to localhost:9002;
    proxy_pass http://localhost:9002;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #proxy_set_header X-NginX-Proxy true;

    # enables WS support
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 999999999;
}
```

### Virtual Host 反向代理

以配置 Nextcloud 为例，首先创建其 Virtul Host 配置文件：

```sh
vim /etc/nginx/sites-available/nextcloud

server {
    #常规 HTTP/HTTPS 端口，用于局域网访问。遵循一端口全局仅配置一次 ipv6=off 规则，例如 openmediavault 的 80 或 443 端口的配置语句已经包含了 `ipv6only=off`，下面就要去除掉对应的 `ipv6only=off`
    listen [::]:80 ipv6only=off;
    listen [::]:443 ipv6only=off ssl http2;
    
    #出于安全考虑，如果该服务需要被公网访问，额外监听一个端口作为转发专用端口，以对接路由器 WAN 侧
    #listen [::]:8443 ipv6only=off ssl http2;

    server_name nextcloud.linhongbo.com;
    
    ssl_certificate /etc/letsencrypt/live/linhongbo.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/linhongbo.com/privkey.pem;
    ssl_verify_client off;
    proxy_ssl_verify off; 
    location / {
        proxy_pass  http://localhost:9096;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

其中，

- `ipv6only=off` 表示监听的 ipv6 socket 既可以处理 ipv6 也可以处理 ipv4 数据包（这是 Linux 的一个新特性，新版 Ningx 默认为 `on`，即在该端口上仅监听 ipv6 数据包）。这个选项可以使配置更简洁，而无需（且不能）额外配置 ipv4 监听语句，形如 `listen 0.0.0.0:80`。但要注意如果有多个 `server` 块监听同一端口，这种写法要求该端口的 `ipv6only=off` 选项**必须**且**只能**出现一次，否则会导致 `Address already in use` 错误（两个端口同时处理 ipv4 请求）或无法处理 ipv4 请求。也就是说由于 openmediavault WEB 界面自身监听的 80 端口和 443 端口（如果启用 HTTPS）在其配置文件中已经配置了 `ipv6only=off`，遵循只能出现一次的规则，你额外的 Virtual Host 配置中 80 和 443 端口就不需且不能再配置 `ipv6only=off` 了。
- `server_name` 指定服务对应的域名
- `ssl_certificate` 和 `ssl_certificate_key` 分别填写先前申请的 Let's Encrypt 证书、私钥文件；
- `proxy_ssl_verify off` 表示关闭 Nginx 到上游服务器（Nextcloud 容器服务）的 SSL/TLS 校验，由于我是反向代理到 Nextcloud 容器的 HTTPS 服务，因此该选项必须。
- `proxy_set_header Host $http_host` 表示反向代理时替换掉 HTTP Header 中的 host 参数，这对 Nextcloud 是必要的，否则会报域名不被信任的错误；
- `proxy_pass https://localhost:9443` 设置被代理容器监听的本地回环地址和端口号；
- `proxy_set_header X-Real-IP $remote_addr` 和 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for` 指示 Nginx 发起反向代理请求时增加原始 IP 字段到 HTTP Header，这对一些依赖源 IP 来认证客户端的 Web 服务非常重要，例如 Emby，由于反向代理时 Nginx 位于 localhost 或局域网，如不设置此参数 Emby 将认为请求是来源于局域网，从而造成安全功能误判。因此建议所有被代理服务配置上此选项。
- `http2` 支持 http2，加速网站加载。注意仅 HTTPS 支持此特性。

将上述配置文件拷贝到 `/etc/nginx/sites-enabled` 目录使能起来，这里应用软链接

```sh
ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/nextcloud
```

命令 Nginx 重新加载配置

```sh
nginx -s reload
```

## 端口转发（公网访问）

最后，为了能在公网访问 openmediavault 上的 WEB 服务，需要在路由器的防火墙规则中增加一条 WAN:8443 -> openmediavault:8443 的 tcp 端口转发规则，这里 WAN 域端口选择 `8443` 而非 HTTPS 默认的 443 端口是因为该端口已运营商防火墙封锁了。此外由于 80 端口亦被封锁，博主不再配置 HTTP（80）端口的转发规则，因为不会带来任何访问上的裨益（网址仍要指定端口才能访问）。

## VPN

VPN 可以在公网和家庭局域网之间创建隧道，从而容易访问内网的各项服务。很多路由器都提供了这项功能。如果你想使用 Shadowsocks 来替代 VPN，可以参考博主的这篇文章：[OpenWrt 安装 Shadowsocks Server](https://linhongbo.com/posts/shadowsocks-server-on-openwrt/)

## 常见问题

### 反向代理性能差

默认配置，Nginx 会将后端请求缓存起来（如果缓存空间不够还会写入本地文件）再发送给客户端，这样就会对 Emby 等大流量数据流场景形成性能瓶颈，导致卡顿现象。

解决办法是在对应域名的 `server` 块中加入如下一行配置（建议在 `nginx.conf` - `http` 中全局配置）

```
proxy_buffering off;
```

这样，Nginx 获取到后端响应就会立即传送给客户端。
