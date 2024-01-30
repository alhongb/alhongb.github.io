---
layout: post
title: 使用 Repo 下载 AOSP 源代码
categories: [Android]
---

本指导记录了在 Windows 下（除了安装外，其余章节 Linux 同样适用）使用 Repo 工具的过程，包括安装 Repo、配置代理、同步全部或部分 Android Open Source Project（APPS）工程源代码到本地，可方便 Android 系统研究人员后续使用 Source Insight 等工具本地查阅代码。

## 安装 Repo（Windows）

### 安装 Python

repo 是 Python 脚本，依赖于 Python。前往 https://www.python.org/downloads/ 下载安装 Python 3，安装过程注意勾选创建 PATH 环境变量，使 Python 程序目录添加到操作系统环境变量中。
 
### 安装 Git
 
前往 https://git-scm.com/downloads 下载安装 Git，在 Windows 上 Git 还提供了一个 bash 模拟器，称为 Git Bash。

### 安装 Repo

先在用户目录（`%USERPROFILE%`）下创建  `bin` 目录，然后手动下载  https://storage.googleapis.com/git-repo-downloads/repo 到该目录，最后将 `bin` 目录添加到 Windows PATH 环境变量中。

## 配置代理（可选）

在公司内网，可能需要额外配置代理才能让 git 和 repo 连接外网。

### Git 代理配置

编辑 `%USERPROFILE%/.gitconfig`（Windows）或 `~/.gitconfig`（Linux）

```
[http]
    proxy = http://user:password@example.com:8080
[https］
    proxy = http://user:password@example.com:8080
```

注意：连接内网地址时需要关闭代理。

### Repo 代理配置

编辑 `%USERPROFILE%/.bash_profile`（Windows）或 `~/.bash_profile`（Linux），增加如下代理配置：

```sh
export http_proxy=http://user:password@example.com:8080
export https_proxy=$http_proxy
export ftp_proxy=$http_proxy
```

完成上述代理配置后，bash（或 git bash）中运行的 Git 和 Repo 命令就会以代理的方式连接服务器了。

## 创建 AOSP 源代码根目录

```sh
mkdir AOSP
```

## 在源代码根目录初始化 Repo 客户端

在源码根目录执行 `repo init` 获取最新版本的 Repo，并通过指定清单文件来指定 Android 源代码中包含的各个代码库位于工作目录中的什么位置：

```sh
repo init -u https://android.googlesource.com/platform/manifest
```

上面命令会在 AOSP 目录创建 `.repo` 目录，包含最新版本 Repo 以及清单等文件。

如果因不明原因无法下载 Repo，可以用如下命令手动获取：

```sh
mkdir .repo
cd .repo
git clone https://gerrit.googlesource.com/git-repo
mv git-repo repo
```

## 同步源代码

AOSP 由大量项目组成，包含数百个 git 仓库，代码量十分巨大（因此 Google 开发 repo 工具）。同步源代码的方式有两种：

### 同步整个 AOSP 源代码

```sh
repo sync
```

### 同步 AOSP 子工程源代码

通过指定工程名来来执行 `repo sync`，可以仅仅同步感兴趣的那部分源代码，并且在本地保持 AOSP 目录结构：

```sh
repo sync project_name1 projuect_nameN
```

其中工程名可通过查阅 .repo/manifest/default.xml 得到，例如 `platform/frameworks/base`

## 切换不同 AOSP 分支

### 1. 查看可切换的分支

```sh
cd .repo/manifests git branch -a | cut -d / -f 3
```
### 2. 切换分支 (以android-7.0.0_r1为例)  

```sh
repo init -u https://android.googlesource.com/platform/manifest -b android-7.0.0_r1
```

### 3. 同步代码 

```sh
repo sync
```

如果本地版本库中的源代码有一些改动，执行上述命令后，会出现部分文件的提交无法合并的提示，此时使用下面的操作命令：

```sh
repo forall -c git reset --hard
repo init -u https://android.googlesource.com/platform/manifest -b android-7.0.0_r1
repo sync
```

## 下载 Android Kernel 代码

Android Kernel 并不包含在 AOSP 中，需要单独下载。

根据[官方链接](https://source.android.com/setup/build/building-kernels#downloading) ，不同设备类型有不同的 kernel，有的 kernel 包含不止一个仓库，但有的 kernel （例如 common kernel，通用内核）则只有一个仓库，因此需要用 repo 工具来保证拉取到正确的仓库。

因此只拉取通用 kernel 的话，直接 git clone 即可：

```
git clone https://android.googlesource.com/kernel/common
```

注意：Windows 系统会报 `error: invalid path 'include/soc/arc/aux.h'` 的错误，原因是 Windows 文件系统不支持 `aux` 这个文件名，解决办法：前往 https://github.com/aosp-mirror/kernel_common 直接下载 zip

其他 kernel 的下载指令，请参考官方链接。

## 常见问题

- 添加了环境变量，但 Git Bash 仍然报找不到 Python 的错误（Windows）

```
/bin/env: python: No such file or directory
```

这种情况在 Git Bash 中执行 `echo $PATH` 会发现 Python 不在 PATH 变量中，解决办法是重启电脑以及重装 Git，让 Git 重新读取 Windows 的环境变量。

- 如果 Git Bash 执行 repo init 出现 PermissionError 的错误，以管理员身份运行即可
