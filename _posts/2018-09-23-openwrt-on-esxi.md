---
title: ESXi 安装&配置 OpenWrt
categories: [Tutorial, OpenWrt]
tags: [ESXi, OpenWrt]
seo:
  date_modified: 2020-04-06 22:23:17 +0800
---

基于 ESXi X86 虚拟机设备构建 OpenWrt 路由器是一种兼顾性能和敏捷性的软路由解决方案。本文详细介绍了如何在 vSphere ESXi 6.7 下部署最新版本的 OpenWrt。

## 制作 OpenWrt ESXi 镜像

点击链接进入官方网站，在你的 Windows 电脑上安装好 ESXi 镜像制作工具 [StarWind V2V Converter](https://www.starwindsoftware.com/converter)。

点击链接下载最新版本的 [Stable Release OpenWrt 镜像](https://downloads.openwrt.org/)。ESXi 一般运行在 X64 平台，因此我们选择 x86/64 目标平台的镜像文件，文件系统选择 ext4。例如：[https://downloads.openwrt.org/releases/18.06.1/targets/x86/64/openwrt-18.06.1-x86-64-combined-ext4.img.gz](https://downloads.openwrt.org/releases/18.06.1/targets/x86/64/openwrt-18.06.1-x86-64-combined-ext4.img.gz)

解压缩包后，运行 StarWind V2V Converter 工具 - `Local file` - 选择此前下载好的镜像文件 - `VMware ESX server image` - `Next >` - `Next >`，最后生成两个 ESXi 专用镜像文件：

- `openwrt-18.06.1-x86-64-combined-ext4.vmdk`
- `openwrt-18.06.1-x86-64-combined-ext4-flat.vmdk`

## 上传镜像

进入 ESXi 系统的 Web 管理页面，点击 `存储` - `数据存储浏览器` - `创建目录`，创建一个存储 OpenWrt 镜像和其它配置文件的目录，这里我们将其命名为 `OpenWrt` 

![ESXi 创建数据存储目录](/assets/img/post/2018/2018-09-23_215948.png)

上传先前创建的 2 个镜像文件到该目录。上传成功后两个镜像文件会呈现为一个硬盘

![ESXi 上传镜像](/assets/img/post/2018/2018-09-23_220148.png)

## 部署端口组

创建 OpenWrt 虚拟机之前，首先规划好它的网络架构。

假设运行 ESXi 的物理主机有 N（N>1） 个硬件网络适配器，那么通常的做法是将 1 个物理网口用于 WAN，其余的 （N -1 个）物理网口用于构建 LAN。博主的主机有 4 个物理网络适配器，下文以此为例。

**注意：**ESXi 初始的虚拟交换机、端口组请保持默认配置，谨慎编辑，否则你将无法远程连接到 ESXi

另外为了帮助读者理解原理，先简要介绍 ESXi 的虚拟网络架构：

```
VMs（虚拟机）
   ↓↑
Port Groups（端口组）
   ↓↑
vSwitch（虚拟交换机）
   ↓↑ 
Physical NICs（硬件网络适配器）
```

上面是各虚拟网络组件的层级与数据流图，其中

- 端口组

ESXi 并非直接连接到 `vSwitch`（虚拟交换机），而是连接到再划分的端口组，并在虚拟机内部实体化为相应的虚拟网络适配器。端口组是软件逻辑实现，因此一个虚拟交换机上可以划分多个端口组，进而可以实现 VLAN 功能：在 ESXi 的管理界面中为不同的端口组分配不同的 `VLAN ID` 即可。

如果把 `vSwitch` 类比于物理世界中的交换机，那么端口组则类似于为交换机上的一组端口

- 虚拟交换机

可以类比物理世界的交换机来理解

- 硬件网络适配器

虚拟网络只有链接至硬件网络适配器才能与物理世界通讯。如果某个虚拟交换机没有链接至任何一个硬件网卡，那么其上的虚拟机只能与连接到该虚拟交换机的其他虚拟机通信，而不能与外界通信，相当于组建了一个虚拟的内网。

下面正式开始部署端口组。首先创建三个新的虚拟交换机，保持默认配置。如下图所示，含 ESXi 初始的 `vSwitch0` 在内，共有四个虚拟交换机，将来分别对应物理机上的四个网口。

![ESXi 虚拟交换机](/assets/img/post/2018/esxi-vswitch.png)

点击编辑虚拟交换机按钮，为新创建的 `vSwitch` 分别链接上行链路（物理网络适配器），从而与主机上的物理网口链接起来

![虚拟交换机添加上行链路](/assets/img/post/2018/2018-09-23_230513.png)

下一步，考虑到 ESXi 虚拟网络未来拓展性（方便、安全地部署其他 VMs），以默认配置创建 `VM Network1` 、`VM Network2` 、`WAN Network` 三个端口组，分别连接到之前新创建的 3 个虚拟交换机上。这项操作目的是为未来其他的 VMs 预留默认安全配置的端口组（区别于后面专为 OpenWrt VM 创建的开启混杂模式端口组），本身和搭建 OpenWrt VM 没有关系。

下一步，创建 `OpenWrt LAN0` 、`OpenWrt LAN1` 、`OpenWrt LAN02`、`OpenWrt WAN` 四个端口组，其中 LAN 系列端口组开启`混杂模式`、`MAC 地址修改`、`伪传输`三个安全选项，而 WAN 端口组保留默认配置，即上述选项全部关闭。

![ESXi 添加端口组](/assets/img/post/2018/2019-01-15_204948.png)

![ESXi 端口组安全选项](/assets/img/post/2018/2019-01-15_205334.png)

最终网络拓扑如图所示

![ESXi OpenWrt 网络逻辑架构](/assets/img/post/2018/ESXi-OpenWrt-Architecture-1.png)

其中 OpenWrt 虚拟机通过 `OpenWrt LAN0` ~ `OpenWrt LAN2`和 `OpenWrt WAN` 四个端口组连接到虚拟交换机 `vSwitch0` ~ `vSwitch3` 进而连通 `VMNIC0` ~ `VMNIC3` 四个物理网口。由于 `OpenWrt LAN` 系列端口组开启了混杂模式，具备监控虚拟网络流量的能力，当其中某个虚拟交换机收到 MAC 帧时，会将此帧转发复制到 OpenWrt `br-lan`（用于 LAN 口设备桥接的虚拟网络，使多个虚拟或物理网络接口的行为好像他们仅有一个网络接口一样），然后再由内部网桥系统决定如何处理（转发到某个 LAN 口还是路由至 WAN 上）。如果 OpenWrt LAN 端口组不开启混杂模式，由于虚拟交换机不具备 MAC 地址学习能力，将丢弃需要转发的 MAC 帧，OpenWrt 将无法接收和转发数据流量。

进一步解释 `混杂模式` 作用：因为虚拟交换机不像传统物理交换机，它不具备 MAC 学习功能力：`vSwitch` 所有接入端口的设备（VMs）是预先配置好的，虚拟交换机它知道这些接入端口组的接口的 MAC 地址，而不需要根据数据包的源 MAC 来更新 MAC 与 PORT 的映射。因此，在混杂模式未开启情况下，如果目标 MAC 不属于该虚拟交换机，那么虚拟交换机将丢弃该数据包。虚拟交换机或端口组开启混杂模式后，所属的 PORT 将收到虚拟交换机上 VLAN 策略所允许的所有流量。这种特性可用来监控虚拟网络流量。

至此，完成端口组的配置。这些端口组后续将被 OpenWrt 虚拟机内的虚拟网络适配器使用。

## 创建 OpenWrt 虚拟机

登入 ESXi Web 管理面，`虚拟机` - `创建/注册虚拟机`

创建新虚拟机

![ESXi 创建虚拟机](/assets/img/post/2018/2018-09-23_214705.png)

![ESXI 创建虚拟机](/assets/img/post/2018/ESXI-create-OpenWrt-vm.png)

我的配置如下 `2 CPU` `256 RAM` 移除默认的硬盘，点击添加硬盘，添加一个现有硬盘。选择先前上传的 `openwrt-18.06.1-x86-64-combined-ext4.vmdk`，硬盘选项保持默认。

![ESXI OpenWrt 添加硬盘](/assets/img/post/2018/2019-01-15_212759.png)

添加网络适配器

![ESXi OpenWrt 添加端口组](/assets/img/post/2018/2019-01-15_213112.png)

编辑虚拟机，将引导选项改为 `BIOS`

![ESXi 修改虚拟机引导方式](/assets/img/post/2018/2018-09-24_000618.png)

**注意：**如果使用默认的 EFI 引导，OpenWrt 将无法启动，报如下错误 `Attempting to start up from: → EFI Virtual disk (0.0) ... unsuccessful. → EFI Network... unsuccessful`

打开虚拟机电源，OpenWrt 正常启动。

## 配置 OpenWrt

现在，要让 OpenWrt 虚拟机内的 4 个虚拟网络适配器正确连接到先前部署的 WAN/LAN 端口组。

查看并记住 OpenWrt 虚拟机连接到 `OpenWrt WAN` 的虚拟网络适配器 MAC 地址

![ESXi OpenWrt Mac 地址](/assets/img/post/2018/2019-01-15_214132.png)

在 ESXI 虚拟机控制台执行以下命令，查找上述 MAC 地址对应的网络适配器

```sh
ifconfig -a | less
```

如图所示，可以看到是 `eth2`

![ESXi OpenWrt ifconfig](/assets/img/post/2018/2019-01-15_214428.png)

**提示：**如果 OpenWrt 控制台反复回显标准输出，导致控制台内容被覆盖，执行 `/etc/init.d/network stop` 可减少标准输出

编辑 /etc/config/network 按照对应关系，将 `eth2` 配置为 `wan` 接口，`eth0`、`eth1`、`eth3` 分配为 `lan` 接口，并为 `lan` 开启 bridge，使 LAN 下所有的物理网口桥接到一起。

```sh
vim /etc/config/network

config interface 'loopback'
option ifname 'lo'
option proto 'static'
option ipaddr '127.0.0.1'
option netmask '255.0.0.0'

config globals 'globals'
option ula_prefix 'fdaf:b952:d594::/48'

config interface 'lan'
option type 'bridge'
option ifname 'eth0 eth1 eth3'
option proto 'static'
option ipaddr '192.168.1.1'
option netmask '255.255.255.0'
option ip6assign '60'
option _orig_ifname 'eth3'
option _orig_bridge 'true'

config interface 'wan'
option ifname 'eth2'
option proto 'dhcp'

config interface 'wan6'
option ifname 'eth2'
option proto 'dhcpv6'
```

编辑完后执行以下命令使配置生效

```sh
/etc/init.d/network reload
```

最后，电脑网线连接到主机的任意一个 lan 物理网口，浏览器输入 192.168.1.1 即可访问 OpenWrt 虚拟机的 LuCI 管理面。

![OpenWrt LuCI](/assets/img/post/2018/2018-09-24_004438.png)

相关文章：[OpenWrt 常用网络配置]({% post_url 2018-09-22-openwrt-usual-configuration %})
