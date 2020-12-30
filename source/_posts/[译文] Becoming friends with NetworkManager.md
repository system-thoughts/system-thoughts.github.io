---
title: 【译文】 Becoming friends with NetworkManager
date: 2020-12-30T21:06:21+08:00
tags: [RedHat, NetworkManager, Network]
categories:
  - translation
  - Linux tools
---

在使用RedHat系OS进行网络管理时，网上很多CentOS7网络配置的教程往往都会建议关闭Networkmanager使用network.service的方式进行网络配置：1. 配置`/etc/sysconfig/network-scripts/ifcfg-<net device>.cfg`；2. `systemctl restart network`重启网络生效。 3. 建议关闭NetworkManger，防止和network.service冲突。
然而network-scripts在RHEL8中已经默认不安装了，RHEL8推荐使用NetworkManager的`nmcli`命令进行网络配置。由于`bridge-utils`项目宣布[弃用](https://git.kernel.org/pub/scm/linux/kernel/git/shemminger/bridge-utils.git/commit/?id=ab8a2cc330253321be7bc69dea88bfaa3d48415e),RHEL8已经无法使用`brctl`命令创建网桥，需要依赖NetworkManager的`nmcli`命令创建。
RHEL8开始，越来越多的网络管理手段貌似只能通过NetworkManager进行，然而RedHat推行多年的NetworkManger一直不温不火，究竟为何？RedHat工程师Francesco Giudici写的本篇文章就是向公众推荐NetworkManager，并试图解释NetworkManager背后的原理。

---

> 原文链接： https://www.redhat.com/sysadmin/becoming-friends-networkmanager

您对Linux主机自动配置网络感到惊讶吗？很有可能这是NetworkManager帮您完成这项任务。NetworkManager是当今Linux发行版中最为广泛使用的网络配置守护程序。
然而，您是否禁用了NetworkManager，并且想知道为什么您首选的Linux发行版没有使用旧的IP工具作为默认的网络配置方法？您是否认为NetworkManager仅适用于WiFi？如果您有上述疑问这篇文章或许适合您。接下来请抛开偏见，给NetworkManager一个公平的机会。我敢打赌，读完这篇文章后，您会放下对NetworkManager的成见，甚至喜欢上它。
我会介绍为什么NetworkManager是许多场景下（包括命令行和GUI）的理想选择。随后，我将解释该工具的基本原理（经常被误解）。最后，我将重点介绍每个用户应该掌握关于NetworkManager的一些命令。

## 为什么是NetworkManager？
Linux主机中配置网络的方法有很多，因此您可能会问为什么要专门使用NetworkManager。尽管已经有很多不错的答案，但我想强调一个经常被忽略的要点：NetworkManager允许用户和应用程序同时检索(retrieve)和修改网络的配置，从而确保获得一致且最新的网络视图。
可通过桌面GUI（Gnome、KDE、nm-applet）、文本界面（`nmtui`）、命令行（`nmcli`）、文件和Web控制台（`Cockpit`）多种方式配置此工具。NetworkManager为应用程序提供了API：D-Bus接口、libnm库。其他网络配置工具都没有这么灵活。
## NetworkManager的原理
为了理解和掌握NetworkManager，您首先需要了解其基础配置原理。以下摘自man NetworkManager：
> “尝试使网络配置和操作尽可能轻松，自动”

这与标准Unix守护程序原理截然不同。传统的Unix守护程序通常要求用户通过配置文件显式地提供配置，然后重启服务。没有配置，守护程序将不执行任何操作。
相反，当只有部分配置或没有配置时，NetworkManager会检查可用设备并尽力提供主机的网络连接。NetworkManager的目标是满足所有人的需求：包括寻找“可以正常工作的网络”的普通用户到需要完全控制主机网络的高级网络管理员。
NetworkManager希望高级网络管理员提供自己的配置以避免网络自动配置。通过配置Linux守护程序来避免执行某些操作（自动化配置）确实不常见，但这看起来是适应任何场景的合理方法。
## 基本的NetworkManager概念
NetworkManager的配置基于设备(device)和连接(connection)的概念。设备映射一个网络接口，大致相当于您执行`ip link`命令看到的接口。每个设备跟踪：
* 是否由NetworkManager管理
* 设备的可用连接
* 设备上的活动连接（如果有）

连接表示要在设备上应用的完整配置，就是个属性列表。 属于同一配置区域的属性被划分为设置组（例如，ipv4设置组属性，包含地址，网关和路由）。
在NetworkManager中配置网络就是简单地激活设备上的连接：设备随后会被配置为连接中的所有属性。
尽管前面提及的多个工具都可配置NetworkManager，但现在我们将重点放在NetworkManager自带的命令行工具`nmcli`上。

> 注意：`nmcli`程序具有高级的自动补全功能。确保安装了`bash-completion`软件包以利用此优势。

## nmcli基础
让我们看下使用`nmcli`处理网络的各个方面。
### 设备
列出NetworkManager检测到的设备：
```bash
$ nmcli device
DEVICE   TYPE       STATE           CONNECTION
enp1s0   ethernet   connected       ether-enp1s0
enp7s0   ethernet   disconnected    --
enp8s0   ethernet   disconnected    -- 
```
从输出中可见，NetworkManager已检测到系统中的三个以太网设备。仅第一个设备`enp1s0`上有活动连接（表示已配置）。
如果您希望NetworkManager一段时间内不再管理一个设备，无需将其关闭，仅需暂时取消管理(unmanage)该设备即可：
```bash
$ nmcli device set enp8s0 managed no
$ nmcli device
DEVICE   TYPE       STATE          CONNECTION
enp1s0   ethernet   connected      ether-ens3
enp7s0   ethernet   disconnected   --
enp8s0   ethernet   unmanaged      --
```
此更改不是持久化的，重启不生效。
查询每个设备IP配置最简单的方法就是不带参数的执行`nmcli`命令：
```bash
$ nmcli
enp1s0: connected to enp1s0
      "Red Hat Virtio"
      ethernet (virtio_net), 52:54:00:XX:XX:XX, hw, mtu 1500
      ip4 default
      inet4 192.168.122.225/24
      route4 0.0.0.0/0
      route4 192.168.122.0/24
      inet6 fe80::4923:6a4f:da44:6a1c/64
      route6 fe80::/64
      route6 ff00::/8

enp7s0: disconnected
      "Intel 82574L"
      ethernet (e1000e), 52:54:00:XX:XX:XX, hw, mtu 1500

enp8s0: unmanaged
      "Red Hat Virtio"	
      ethernet (virtio_net), 52:54:00:XX:XX:XX, hw, mtu 1500
```
### 连接
列出可用连接：
```bash
$ nmcli connection
NAME                 UUID                                  TYPE       DEVICE
ether-enp1s0         23e0d89e-f56c-3617-adf2-841e39a85ab4  ethernet   enp1s0
Wired connection 1   fceb885b-b510-387a-b572-d9172729cf18  ethernet   --
Wired connection 2   074fd16d-daa6-3b6a-b092-2baf0a8b91b9  ethernet   --
```
从输出可见，唯一的活动连接是应用于设备`enp1s0`的`ether-enp1s0`。也存在其他两个连接，但是它们不处于活动状态。
要停用连接，即取消配置关联的设备，只需指示NetworkManager断开连接即可。 例如，要停用ether-enp1s0连接：
```bash
$ nmcli connection down ether-enp1s0
```
要重新激活它，即重新配置设备：
```bash
$ nmcli connection up ether-enp1s0
```
要查看特定连接的详细信息，我们应该检查连接的属性：
```bash
$ nmcli connection show ether-enp1s0
connection.id:                     ether-enp1s0
connection.uuid:                   23e0d89e-f56c-3617-adf2-841e39a85ab4
connection.stable-id:              --
connection.type:                   802-3-ethernet
connection.interface-name:         enp1s0
connection.autoconnect:            yes
connection.autoconnect-priority:   -999
connection.autoconnect-retries:    -1 (default)
connection.auth-retries:           -1
connection.timestamp:              1559320203
connection.read-only:              no
[...]
```
连接属性的列表很长，并且按设置分组。 实际上，每个属性都被指定为`setting_name.property_name`。我们重点介绍一些属于连接和IPv4设置的基本属性：
| 属性 | 描述 | 别名 |
| --- | --- | --- |
| connection.id | 连接名称(nmcli connection中输出) | con-name |
| connection.uuid | 全局唯一标识符UUID，唯一标识连接 | (none) |
| connection.type | 连接类型 | type |
| connection.interface-name | 将连接绑定到特定设备，只能在该设备上激活该连接 | ifname |
| connection.autoconnect | 连接是否应该被自动激活 | autoconnect |
| ipv4.method | 连接的IPv4方法：自动(auto)，禁用(disabled)，链接本地(link local)，手动(manual)或共享(shared) | (none) |
| ipv4.addresses | 连接的静态IPv4地址 | (none) |

> 请注意，少数常用属性具有别名，即可以用来代替完整设置的短名称。上表在第三列中列出了别名。 而且，所有nmcli命令都可以被截断，且能执行相同的操作。 例如，要关闭ether-enp1s0连接的自动连接，我们可以将上面的modify命令缩短为：
`$nmcli c m ether-ens1s0 autoconnect no`

`autoconnect`属性控制连接的自动激活。如果启用了它（`= yes`，默认值），`interface-name`设备就绪状态且其上无活动连接时，NetworkManager会自动激活该连接。 如果将`ipv4.method`设置为`auto`，则可以通过DHCP配置IPv4。




