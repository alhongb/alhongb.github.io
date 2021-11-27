---
layout: post
title: CentOS 7 Apache 配置 Virtual Host （虚拟主机）
categories: [Web]
tags: [Apache]
seo:
  date_modified: 2020-03-02 22:56:24 +0800
---

## 简介

Apache 是世界上最流行的网页服务器，其功能和组件被分解为不同的单元与功能，其中基础的配置单元是站点（域名）。Apache 用 Virtul Host 表示站点，通过支持多 Virtual Host ，便能够在单个服务器上托管多个站点，共享服务器资源，而且保持互相独立。这里要注意子域名和一级域名一样，都可作为不同的 Virtual Host 进行配置。

这篇教程，将介绍在 CentOS 7 VPS 上如何配置多个 Apache Virtual Host。

## 提前准备

要使用此教程，你至少要有一个可用的域名（包括子域名）。下面的教程假设你有 example.com 和 example2.com 两个域名需要配置 Apache Virtual Host。

## Step 1 - 创建目录结构

首先，我们要为不同域名（站点）分别创建存放网页数据的目录。

```sh
sudo mkdir /var/www/example.com/html
sudo mkdir /var/www/example2.com/html
```

**注意：** `example.com`、`example2.com` 为你想配置的站点，这里引入 `html` 二级目录作为 Virtual Host 的根目录，是为了与日志文件等隔离开，如果你要创建的站点是 WordPress、Nextcloud 等服务，可将 `html` 变更为相应的服务名称。

CentOS 7 未引入  sites-enabled 、sites-available 目录结构，我们需要为 Apache 创建 这两个目录（这样有利于后续灵活直观地管理 Virtual Host）

```sh
sudo mkdir /etc/httpd/sites-available
sudo mkdir /etc/httpd/sites-enabled
```

**提示：**你应该将可用 Virtual Host 配置文件统统存储在 site-available 目录下，而在 sites-enabled 目录下配置已启用的 Virtual Host。

现在，让 Apache 加载 sites-enabled 目录下的配置文件。编辑 `http.conf`，尾部增加一行， `IncludeOptional sites-enabled/*.conf`

```sh
sudo vim /etc/httpd/conf/httpd.conf 

...
IncludeOptional sites-enabled/*.conf
```

## Step 2 - 创建新 Virtual Host 配置文件

创建 `example.com.conf` 文件，指定 `/var/www/example.com/html` 为 DocumentRoot

```sh
sudo vim /etc/httpd/sites-available/example.com.conf

<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/example.com/html
    ErrorLog /var/www/example.com/error.log
    CustomLog /var/www/example.com/requests.log combined
</VirtualHost>

<Directory /var/www/example.com/html>
    Options -Indexes -FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>

# 某些服务如 Nextcloud 需要覆写 .htaccess ，那么使用下面的配置
#<Directory /var/www/example.com/nextcloud>
#    AllowOverride All
#    Require all granted
#</Directory>
```

这里 `<VirtualHost *:80>...</VirtualHost>` 表示监听 80 端口的一个 Virtual Host， `ServerName` 指明 Virtual Host 的绑定的域名； `DocumentRoot` 指定 Virtual Host 的根目录（ 意味着 example.com 对 html 目录外的访问都将被禁止）；`ErrorLog` 、`CustomLog` 指定日志存储目录。

出于安全考虑，我们在 `<Directory /...;</Directory>` 标签中使用 `-Indexes` 禁止目录浏览；使用 `-FollowSymLinks` 禁止符号链接逃逸 document root；`AllowOverride None` 不允许 .htaccess 覆写；`Require all granted` 允许所有 IP 连接。

**注意：**`<Directory /...>` 应使用绝对路径；新版本 Apache 已不允许 `Options -Indexes FollowSymLinks` 这种既有+/-参数又有非+/-参数的写法，应该统一写成 `Options -Indexes +FollowSymLinks` ，或去掉 `-Indexes` 。

## Step 3 - 使能 Virtual Host 文件

现在，要让先前创建的 Virtual Host 配置文件生效了。在 `site-enabled` 目录下建立 `example.com.conf` 的软连接

```sh
sudo ln -s /etc/httpd/sites-available/example.com.conf /etc/httpd/sites-enabled/example.com.conf
```

重启 Apache ，使配置生效

```sh
apachectl restart
```

## Step 4 - 测试最终结果

在 `/var/www/example.com/html` 放置你要展示的 index.html ，然后使用浏览器访问 `http://example.com` 。

若一切正常，你的页面将正常展示。接下来去完成另一个域名 example1.com 的配置吧。

## Step 5 - 指定默认 Virtual Host （可选）

现在我们配置了多个站点，Apache 将会根据访问不同域名匹配不同的 Virtual Host，一切正常。但是，如果访问的域名与任何 `ServerName` 都不匹配，例如使 IP 访问时将连接哪个 Virtual Host 呢？

配置文件第一个被加载的为默认 Virtual Host。

在本文场景中，我们配置了 `example.com.conf` 和 `example2.com.conf` 两个 Virtual Host，Apache 会首先加载 `example.con.conf` 然后加载 `example2.com.conf` ，依据文件名排序。也就是说，如果使用 IP 访问站点，加载的将会是 example.com 。如果你想让 example2.com 成为默认的站点，可以在将其配置文件命名为 `0-example2.com.conf` 。要自定义默认 Virtual Host，通常的做法是建立一个 `000-default.conf` 的配置文件。
