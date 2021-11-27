---
layout: post
title: 大容量 VPS 搭建 Nextcloud 个人云盘 + aria2 在线下载
categories: [Tutorial, NAS]
tags: [NAS]
seo:
  date_modified: 2020-03-03 21:19:25 +0800
---

## 简介

Nextcloud 是一套用于创建网络硬盘的客户端－服务器开源软件，使用与 Dropbox 相近。每个人都可以在私人服务器上安装并运行它。

aria2 是一款流行的开源下载工具，支持多种协议，包括磁力链接和 BT。

Resillio Sync，你可能已经听说过这款神奇的文件同步软件，在本文中会作为 aria2 BT下载功能的补充。

这篇教程所使用的环境是 CentOS 7 LAMP (Linux, Apache, MySQL, and PHP) 


## 提前准备

- **选购**大硬盘 VPS 并安装 CentOS 7。如果你觉得这篇文章有帮助可以前往[优惠链接](https://billing.hostens.com/?affid=451)购买 Hostens Storage VPS，遵从官网指引订购即可

这里需要提醒，Hostens 机房到国内速度不稳定，如果在意这一点的谨慎购买。博主因为使用 SS 等代理连接，速度很快，可以忽略。

- **安装**Apache 或 Nginx，Hostens VPS 可选的 CentOS 7 系统镜像已带有 Apache，本文直接使用 Apache
- **配置**域名 DNS，并解析至你的 VPS IP

## Step 1 - 安装 PHP7.2

CentOS 7 默认 PHP 版本不满足最低求，你需要启用 EPEL 和 Remi 库。

```sh
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
```

为了使用 `yum-config-manager` 命令，安装 yum-utils

```sh
sudo yum install yum-utils
```

配置 PHP 软件库版本，安装 PHP

```sh
sudo yum-config-manager --enable remi-php72  
sudo yum install php
```

> 截至发稿，可用的最新版本为 7.2，你可以修改为 7.x 以获取其他 PHP 版本，但注意命令执行会有成功提示，否则还是安装默认的低版本 PHP。

查看当前 PHP 版本:

```sh
php -v
```

安装额外的 PHP 模块

```sh
sudo yum install php-mysql php-pecl-zip php-xml php-mbstring php-gd
```

配置 PHP 上传大小限额

首先获取 PHP 配置加载路径：

```sh
php --ini |grep Loaded

Loaded Configuration File: /etc/php.ini
```

修改 `post_max_size`、`upload_max_filesize`

```sh
sudo vim /etc/php.ini

post_max_size = 100M
upload_max_filesize = 100M
```

重启Apache、PHP

```sh
sudo systemctl restart httpd
sudo systemctl restart php7.2-fpm.service
```

## Step 2 - 安装并配置数据库

MariaDB 是社区开发的 MySql 分支，CentOS 7 默认数据软件为 MariaDB。Nextcloud 运行需要一个数据库。

安装 MariaDB

```sh
sudo yum -y install mariadb mariadb-server
```

开机启动 MariaDB

```sh
sudo systemctl start mariadb 
sudo systemctl enable mariadb
```

执行 `mysql_secure_installation` 配置数据库 root 密码及安全选项。如果只是搭建 Nextcloud，后面几个都选择 Y 即可

```sh
sudo mysql_secure_installation

Enter current password for root (enter for none): ENTER
Set root password? [Y/n] Y
Remove anonymous users? [Y/n] Y
Disallow root login remotely? [Y/n] Y
Remove test database and access to it? [Y/n] Y
Reload privilege tables now? [Y/n] Y
```

为 Nextcloud 创建一个数据库

```sh
mysql -u root -p

MariaDB [(none)]> CREATE DATABASE nextcloud;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost' IDENTIFIED BY 'YOURPASSWORD';'nextclouduser'@'localhost' IDENTIFIED BY '你的数据库密码';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> q
```

## Step 3 -安装 Nextcloud 13

前往  Nextcloud 官网下载最新发布的版本

```sh
cd /tmp
wget https://download.nextcloud.com/server/releases/nextcloud-xx.x.x.zip
sudo mkdir /var/www/yourdomain.com/
sudo unzip nextcloud-xx.x.x.zip -d /var/www/yourdomain.com/
sudo cd /var/www/yourdomain/nextcloud
sudo chown -R apache:apache config data apps
```

出于安全考虑，你可以只将 Nextcloud 运行必须的三个文件夹属主修改为 apache，后续更新版本再临时修改 `nextcloud/` 的属主为 apache

## Step 4 - 配置 Apache

让 CentOS 7 Apache2 支持 sites-enabled 目录结构（这样能够使以后站点配置更加灵活直观）。编辑 `http.conf`，尾部增加一行

```sh
sudo vim /etc/httpd/conf/httpd.conf

IncludeOptional sites-enabled/*.conf
```

为你的域名添加为一个 Virtual Host 文件，指向 `nextcloud` 目录

```sh
sudo vim /etc/httpd/sites-enabled/yourdomain.com.conf

<VirtualHost *:80>
        ServerName yourdomain.com
        DocumentRoot /var/www/yourdomain.com/nextcloud
        ErrorLog /var/www/yourdomain.com/error.log
        CustomLog /var/www/yourdomain.com/requests.log combined
</VirtualHost>

<Directory /var/www/yourdomain.com/nextcloud>
        AllowOverride All
        Require all granted
</Directory>
```

**提示：**支持  sites-enabled 、sites-available 目录结构可以让你更方便地配置和管理Virtual Host，你应该将不同 Virtual Host 配置文件统统存储在 site-available 目录下，而在 sites-enabled 目录下配置软链接来映射你想启用的 Virtual Host。Debian 发行版已经支持。

## Sep 5 - 完成 Nextcloud 安装

访问 `yourdomain.com`，进行最后的 Nextcloud 安装。安装过程中填入前面配置的数据库名、数据用户、数据库密码并创建你的Nexcloud管理员账号。至此，Nextcloud 安装完成。

```
Database user: nextclouduser
Database password: 你的数据库密码
Database name: nextcloud
host: localhost
```

## Step 6 - 安装并配置 aria2

安装 aria2。aria2 在各大 Linux 发行版中已可直接安装

```sh
## CentOS 7:
sudo yum install aria2

## Ubuntu:
sudo apt install aria2
```

也采用编译安装的方式，安装最新版本

//TODO: 补充编译安装

配置运行 aria2 守护进程。创建 aria2 配置文件，注意指定你自己的 `rpc-secret`

```sh
sudo vim /etc/aria2.conf

## Basic Options
dir=/home/apache/downloads
input-file=/var/aria2/aria2.session
log=/var/aria2/aria2.log
max-concurrent-downloads=3
max-connection-per-server=8
check-integrity=true
continue=true

## BitTorrent/Metalink Options
bt-enable-lpd=true
bt-max-open-files=16
bt-max-peers=8
dht-file-path=/var/aria2/dht.dat
dht-listen-port=6801
#enable-dht6=true
listen-port=6801
max-overall-upload-limit=0K
seed-ratio=1.0

## RPC Options
enable-rpc=true
rpc-allow-origin-all=true
rpc-listen-all=true
rpc-listen-port=6800
rpc-secret=<你的口令>
#rpc-secure=true
#save-session-interval=2
#force-save=true
check-certificate=false
rpc-save-upload-metadata=true

## Advanced Options
daemon=true
#enable-mmap=true
log-level=warn
file-allocation=none
max-overall-download-limit=0K
save-session=/var/aria2/aria2.session
always-resume=true
split=4
min-split-size=10M

## Pan.baidu.com user agent
#user-agent=netdisk;7.8.1;Red;android-android;4.3
```

创建 aria2 运行所需的文件

```sh
sudo mkdir /var/aria2
sudo cd /var/aria2
sudo touch aria2.session dht.dat aria2.log
sudo chown apache:apache aria2.session dht.dat aria2.log
```

创建 aria2下载目录。这个目录要可以被 Nextcloud 访问，因此该目录属主必须是 apache（其他 Linux 发行版也可能是 `www-data`）

```sh
sudo mkdir -p /home/apache/downloads
sudo chown apache:apache -R /home/apache/downloads
```

创建 aria2 的 systemd 服务文件。记住你需要以 apache 用户运行 aria2

```sh
sudo vim /etc/systemd/system/aria2.service 

[Unit]
Description=Aria2c download manager
Requires=network.target
After=network.target

[Service]
Type=forking
User=apache
RemainAfterExit=yes
ExecStart=/usr/bin/aria2c --conf-path=/etc/aria2.conf
ExecReload=/usr/bin/kill -HUP $MAINPID
RestartSec=1min
Restart=on-failure

[Install]
WantedBy=multi-user.target 
```

启动 aria2 并使其开机启动

```sh
sudo systemctl restart aria2.service
sudo systemctl enable aria2.service
```

使用 AriaNg 远程连接 aria2。AriaNg 是 aira2 的一款前端软件，用来控制 aria2 执行下载任务。安装过程十分简单：前往 GitHub 下载最新版本的 AriaNg，解压至 `/var/www/yourdomain.com/nextcloud` 即可。

```sh
cd /tmp
wget https://github.com/mayswind/AriaNg/releases/download/0.4.0/aria-ng-0.4.0.zip
unzip aria-ng-0.4.0.zip -d /var/www/yourdomain.com/nextcloud/ariang
```

访问 `yourdomain.com/ariang`，使用先前配置的 rpc-secret 连接。

到这里，你已完成服务器端的全部完成。

## Step 7 - 让你的 Nextcloud 集成 aria2

现在，你的 aria2 能够正常运行了，下载好的文件也将保存到 `/home/apache/downloads` 目录。最后一步，前往 Nextcloud 管理员界面，将 `/home/apache/downloads` 目录添加至 Nextcloud 外部存储，congratulation！

![Nextcloud 外部存储配置](/assets/img/post/2018/Nexcloud-external-storage.png)
