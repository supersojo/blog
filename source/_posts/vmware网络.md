---
title: vmware网络
date: 2020-11-26 20:11:23
tags:
- network
- vmware

categories:
- linux
---

## vmware支持三种网络

* vmnet0 - 桥接网络模式

  虚拟机和物理网口通过桥接方式互联

* vmnet1 - host-only网络模式

  虚拟交换机和外部接口无任何连接

* vmnet8 - nat网络模式

  虚拟机连接到nat网络，虚拟机发出的报文经NAT后从外部接口出去

## 多虚拟机方案

openwrt虚拟机两个接口一个连接nat网络作为WAN口，一个连接host-only网络作为LAN口，其它虚拟机仅连接host-only网络。可以开启openwrt的dhcp，这样虚拟机地址自动配置，无需再设置网关。所有虚拟机流量都是经过加密的。