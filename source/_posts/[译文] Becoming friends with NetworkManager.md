---
title: [译文] Becoming friends with NetworkManager
date: 2020-12-30T21:06:21+08:00
tags: [RedHat NetworkManager Network]
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
然而，您是否禁用了NetworkManager，并且想知道为什么您首选的Linux发行版没有使用旧的IP工具作为默认的网络配置方法？ 您是否认为NetworkManager是仅适用于WiFi”？ 好吧，这篇博客文章也适合您。 跟随几分钟，留下偏见，并给此工具一个公平的机会。 我敢打赌，您会放下对NetworkManager的成见，甚至可以成为朋友。
在本文中，我会介绍为什么NetworkManager是许多场景下（包括命令行和GUI）的理想选择。 接下来，我将解释该工具的基本原理（经常被误解）。 最后，我将重点介绍每个用户应该掌握NetworkManager的一些命令。

## 为什么是NetworkManager？
在Linux主机上配置网络有多种方法，因此您可能想知道为什么要专门使用NetworkManager。 尽管有很多不错的答案，但我想强调一个经常被忽略的要点：NetworkManager允许用户和应用程序同时查询和修改网络的配置，从而确保获得一致且最新的网络视图。
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

