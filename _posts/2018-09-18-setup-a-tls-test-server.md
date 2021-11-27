---
layout: post
title: 搭建 SSL/TLS 测试服务器
categories: [Web]
tags: [TLS]
seo:
  date_modified: 2020-03-02 22:56:24 +0800
---

本文介绍了在 Ubuntu 18.04 环境下利用 OpenSSL 的 s_server 命令搭建 SSL/TLS 测试服务器的方法，可用于测试 Chrome 浏览器下的各种 SSL/TLS 错误。

## 创建私有 CA 和服务器证书

以 Chrome `NET::ERT_CERT_WEAK_KEY` 错误为例，要构造此场景，需要服务器证书的密钥长度小于 1024 bit，我的方法是先生成私有 CA， 然后签发相应密钥长度的服务器证书。

**注意：**不推荐直接使用自签名证书，在Chrome 浏览器上会遇到各种问题；文章发布时 Chrome 最新稳定版本为 M69。

### 创建一个私有 CA

首先，生成服一个 CA 私钥 `rootCA.key`，命令执行过程中需要输入一个密码来保护私钥

```sh
mkdir selfCA
cd selfCA
openssl genrsa -des3 -out rootCA.key 2048
```

生成 CA 证书 `rootCA.crt`

```sh
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt
```

**注意：**命令执行过程中需要按要求录入多个证书信息字段，包括 `Country Name, Organization Name, Common Name` 等等，这些信息是用于标识创建的 CA，而非你的域名。

### 签发服务器证书

创建 SSL 服务器私钥 `selfsigned.key` 以其证书签名请求文件 `server.csr`，后者用于向创建的 CA 请求证书

```sh
openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout selfsigned.key
```

**注意：**最重要的字段是 `Common Name (e.g. server FQDN or YOUR name)`, 你可以使用 `localhost` ，IP 地址或实际的域名，例如 `xxx.example.com`；`challenge` 留空。

创建一个 `v3.ext` 文件，这里用于给服务器证书添加“`使用者可选名称`”扩展字段，以解决 Chrome 下“`Certificate - Subject Alternative Name Missing`”SSL/TLS 错误。内容如下


```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

}[alt_names]
DNS.1 = xxx.example.com
```

最后使用之前创建的 CA 签发服务器证书 `selfsigned.crt`

```sh
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out selfsigned.crt -days 1000 -sha256 -extfile v3.ext
```

**注意：**整个过程中我们为 CA 和服务器分别创建了密码学材料，从密码学上看二者本质上没有区别 — 它们对应的都是私钥、公钥和证书，只是前者用来签发证书，后者用来作 SSL/TLS 服务器凭证，仅仅用途不同而已。

## 运行 SSL/TLS 测试服务

有了服务器证书和私钥，就可以运行 Web 服务器了。OpenSSL 的 s_server 命令已经实现了一个基本的 SSL/TLS 服务，可监听并接受指定端口的 SSL/TLS 连接，我们直接使用此命令启动 Web 服务器。

使用先前生成的服务器证书和私钥启动 s_server

```sh
openssl s_server -cert selfsigned.crt -key selfsigned.key -cipher "ALL:@SECLEVEL=0" -www -accept 4433
```

其中 `-accept 4433` 指定监听 4433 端口; `-www` 表示向连接的客户端返回 SSL/TLS 握手信息，以 HTML 格式（如果需要响应指定 HTML 文件，使用 `-WWW`，然后以 http://yourdomain/page.html 形式访问文件）

到此，你已经可以使用 Chrome 访问测试服务了，但是会出现证书不被系统信任的 SSL 错误，我们还需要将创建的 CA 添加到系统信任证书颁发机构。

## 将 CA 证书添加至操作系统信任列表

### Windows

双击 `rootCA.crt`，选择安装证书，导入到「`受信任的根证书颁发机构`」。

![Windows 10安装根证书](/assets/img/post/2018/cert-manager-Windows10.png)

### **Android**

`系统设置` 里选择“安装证书”或直接点击 `rootCA.crt` 进行安装，安装时选择「`VPN 和应用`」。
