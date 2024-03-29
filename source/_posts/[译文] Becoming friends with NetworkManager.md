---
title: 【译文】 Becoming friends with NetworkManager
date: 2020-12-30T21:06:21+08:00
tags: [RedHat, NetworkManager, Network]
categories: [translation, tools]
headimg: /images/headimg/networkmanager.png
---

网上很多CentOS7网络配置的教程往往都会建议关闭Networkmanager使用network.service的方式进行网络配置：1. 配置`/etc/sysconfig/network-scripts/ifcfg-<net device>.cfg`；2. `systemctl restart network`重启网络生效。 3. 建议关闭NetworkManger，防止和network.service冲突。
然而，network-scripts在RHEL8中已经默认不安装了。RHEL8推荐使用NetworkManager的`nmcli`命令进行网络配置。另外，`bridge-utils`项目宣布[弃用](https://git.kernel.org/pub/scm/linux/kernel/git/shemminger/bridge-utils.git/commit/?id=ab8a2cc330253321be7bc69dea88bfaa3d48415e),在RHEL8中无法使用`brctl`命令创建网桥，需要依赖NetworkManager的`nmcli`命令创建。
RHEL8开始，越来越多的网络管理手段貌似只能通过NetworkManager进行，然而RedHat推行多年的NetworkManger却一直不温不火，究竟为何？RedHat工程师Francesco Giudici写的本篇文章就是向公众推荐NetworkManager，并试图解释NetworkManager背后的原理。

---

> 原文链接： https://www.redhat.com/sysadmin/becoming-friends-networkmanager

您对Linux主机自动配置网络感到惊讶吗？很有可能是NetworkManager帮您完成这项任务。NetworkManager是当今Linux发行版中最为广泛使用的网络配置守护程序。
然而，您是否禁用了NetworkManager，并且想知道为什么您首选的Linux发行版没有使用旧的IP工具作为默认的网络配置方法？您是否认为NetworkManager仅适用于WiFi？如果您有上述疑问，这篇文章或许适合您。接下来，请抛开偏见，给NetworkManager一个公平的机会。我敢打赌，您在读完这篇文章后，会放下对NetworkManager的成见，甚至会喜欢上它。
我会介绍为什么NetworkManager是许多场景（包括命令行和GUI）下的理想选择。随后，我将解释该工具的基本原理（经常被误解）。最后，我将重点介绍每个用户应该掌握关于NetworkManager的一些命令。

## 为什么是NetworkManager？
Linux主机中配置网络的方法有很多，因此您可能会问为什么要专门使用NetworkManager。尽管已经有很多不错的答案，但我想强调一个经常被忽略的要点：NetworkManager允许用户和应用程序同时检索(retrieve)和修改网络的配置，从而确保获得一致且最新的网络视图。
可通过桌面GUI（Gnome、KDE、nm-applet）、文本界面（`nmtui`）、命令行（`nmcli`）、文件和Web控制台（`Cockpit`）多种方式配置此工具。NetworkManager还为应用程序提供了API：D-Bus接口、libnm库。任何其他网络配置工具都没有这么灵活。
## NetworkManager的原理
为了理解和掌握NetworkManager，您首先需要了解其基础配置原理。以下摘自man NetworkManager：
> “尝试使网络配置和操作尽可能轻松，自动”

这与标准Unix守护程序原理截然不同。传统的Unix守护程序通常要求用户通过配置文件显式地提供配置，然后重启服务。没有配置，守护程序将不执行任何操作。
相反，当只有部分配置或没有配置时，NetworkManager会检查可用设备并尽力提供主机的网络连接。NetworkManager的目标是满足所有人的需求：包括寻找“可以正常工作的网络”的普通用户到需要完全控制主机网络的高级网络管理员。
NetworkManager希望高级网络管理员提供自己的配置以避免网络自动配置。通过配置Linux守护程序来避免执行某些操作（自动化配置）确实不常见，但这看起来是适应任何场景的合理方法。
## 基本的NetworkManager概念
NetworkManager的配置基于设备(device)和连接(connection)两大概念。设备映射一个网络接口，大致相当于您执行`ip link`命令看到的接口。每个设备跟踪：
* 是否由NetworkManager管理
* 设备的可用连接
* 设备上的活动连接（如果有）

连接表示要在设备上应用的完整配置，就是一个属性列表。 属于同一设置区域的设置属性被划分为设置组（例如，ipv4设置组属性，包含地址，网关和路由）。
在NetworkManager中，配置网络就是简单地激活设备上的连接：设备随后会被配置为连接中的所有属性。
尽管前面提及的多个工具都可配置NetworkManager，但现在，我们重点关注NetworkManager自带的命令行工具`nmcli`。

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
由输出可见，NetworkManager已检测到系统中的三个以太网设备。仅第一个设备`enp1s0`上有活动连接（表示已配置）。
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

> 请注意，少数常用属性具有别名，即可以用来代替完整设置的短名称。上表第三列显示别名。所有nmcli命令都可以被截断，且能执行相同的操作。 例如，要关闭ether-enp1s0连接的自动连接，我们可以将上面的modify命令缩短为：
`$nmcli c m ether-ens1s0 autoconnect no`

`autoconnect`属性控制连接的自动激活。如果启用了它（`= yes`，默认值），`interface-name`设备处于就绪状态且其上无活动连接时，NetworkManager会自动激活该连接。如果将`ipv4.method`设置为`auto`，则可以通过DHCP配置IPv4。
如果您更喜欢配置静态IP，则将`ipv4.method`设置为`manual`，然后在`ipv4.addresses`属性中设定静态IP地址和子网（以CIDR表示法）。有关所有属性及其含义的完整说明，请参见`nm-settings`手册页（`man nm-settings`）。
使用`nmcli connection Modify`命令更改连接的属性。例如，将`ether-enp1s0`连接更改为静态IPv4地址（10.10.10.1），网关（10.10.10.254）和DNS（10.10.10.254）：
```bash
$ nmcli connection modify ether-enp1s0 ipv4.method manual ipv4.addresses 10.10.10.1/24 \
ipv4.gateway 10.10.10.254 ipv4.dns 10.10.10.254
```
此命令将永久更改连接，但是新设置不会立即应用到设备：下次在该设备上激活连接时才会应用新设置。因此，要立即设置并运行我们的设置，请重新激活连接：
```bash
$ nmcli connection up ether-enp1s0
```
您可能还想阻止NetworkManager自动激活`ether-enp1s0`连接：
```bash
$ nmcli connection modify ether-ens1s0 connection.autoconnect no
```
从此开始，您必须自行激活`ether-enp1s0`连接。

### 使用nmcli创建新连接
现在，是时候使用`nmcli`在NetworkManager中创建新连接了！`nmcli connection`子命令`add`的语法类似于`modify`子命令：
```bash
$ nmcli con add type ethernet ifname enp0s1 con-name enp0s1_dhcp autoconnect no
$ nmcli con
NAME UUID TYPE DEVICE
ether-enp1s0 23e0d89e-f56c-3617-adf2-841e39a85ab4 ethernet enp1s0
enp0s1_dhcp 64b499cb-429f-4e75-a54d-b3fd980c39aa ethernet --
Wired connection 1 fceb885b-b510-387a-b572-d9172729cf18 ethernet --
Wired connection 2 074fd16d-daa6-3b6a-b092-2baf0a8b91b9 ethernet --
```
由于`ipv4.method`属性的默认值为`auto`，未指定任何IPv4属性的情况下，IPv4配置将通过DHCP检索获取。
创建连接时，强制属性取决于connection.type（ethernet，wifi，bond，vpn等）。如果缺少任何强制属性，nmcli将返回错误，并打印缺少属性的名称。
如果您希望获得更具交互性的体验，请在您的`nmcli`命令中添加`--ask`标志。这样，`nmcli`会提示您完成命令所需的内容，而不是直接失败：
```bash
$ nmcli --ask con add
Connection type: ethernet
Interface name [*]: enp0s1
There are 3 optional settings for Wired Ethernet.
Do you want to provide them? (yes/no) [yes] no
There are 2 optional settings for IPv4 protocol.
Do you want to provide them? (yes/no) [yes] no
There are 2 optional settings for IPv6 protocol.
Do you want to provide them? (yes/no) [yes] no
There are 4 optional settings for Proxy.
Do you want to provide them? (yes/no) [yes] no
Connection 'ethernet-enp0s1' (64b499cb-429f-4e75-a54d-b3fd980c39aa) successfully added.
```
可通过`nmcli con edit`访问编辑器模式。此模式提供了内联帮助和更加交互的体验。不带任何参数调用此模式会导致`nmcli`编辑器提示您输入要创建的连接类型。若传入连接名称，编辑器将打开该连接以对其进行修改。

### nmcli备忘单
总结下前面提及的`nmcli connection`子命令：

| 命令 | 参数 | 描述 |
| --- | --- | --- |
| `down` | connection | 断开指定的连接，取消配置关联的设备 |
| `up` | connection | 激活指定的连接 |
| `show` | [connection] | 当不带参数使用时，可以像默认命令一样省略show：它列出了所有可用的连接。 如果提供连接名称或UUID作为参数，则将打印连接属性 |
| `modify` | connection {property_name property_value}... | 改变连接的属性 |
| `add` | connection {property_name property_value}... | 创建具有指定属性的新连接。强制属性取决于连接类型，应始终指定该类型 |

### 接下来？
现在，我们介绍了NetworkManager的基本原理及其基本概念（设备和连接）。我们还看到NetworkManager可以通过多种工具以多种方式管理并发请求。我们了解了如何使用NetworkManager命令行工具`nmcli`显示实际配置,在设备上添加，修改和激活/停用连接。这些知识为您提供了了解和掌握NetworkManager的基础。
NetworkManager可以做地更多。许多功能都应提供单独的博客文章，例如调度程序脚本，连接检查器，拆分的DNS，MAC地址随机化，热点配置和自动配置。因此，请继续关注这里的下一篇博客文章。或者，如果您迫不及待，请开始在NetworkManager手册页中查找！





