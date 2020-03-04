---
title: BIOS和UEFI
date: 2019-05-01 21:18:10
tags:
- bios
- uefi
categories:
- pc
---

最近在某发行版遇到问题，大体情况是修改/etc/default/grub的定制脚本，
在BIOS和UEFI主板的机器上结果不同，后调查发现在UEFI机器上，grub的配置文件
位于Efi System Paition分区，而不是BIOS机器上的/boot/grub2下。

这里分析下BIOS和UEFI启动。

## bios
![ bios ](BIOS和UEFI/bios.png)

1. 上电，bios firmware开始执行，加载MBR（第一扇区）到指定位置(0x7c00)开始执行第一阶段启动代码(bootcode)
> 第一阶段启动代码非常小，不超过1个扇区大小（扇区:512字节）

2. stage 1代码负责加载并执行stage 2的代码
> stage 2代码(grub)功能强大，加载操作系统

3. stage 2加载操作系统，操作系统启动

## uefi
![ uefi ](BIOS和UEFI/uefi.png)

1. 上电，uefi firmware开始执行，加载ESP分区的efi文件执行(解释执行efi byte code)
> efi文件中是efi字节码，uefi固件内置efi解释器，用于执行这些代码

2. efi bootcode加载引导程序(grub)并执行引导程序
> stage 2代码(grub)功能强大，加载操作系统

3. stage 2加载操作系统，操作系统启动

## 区别
在系统启动上，除了1和2差别很大，后边系统启动完全一样了。

## 参考
1. [uefi vs bios](https://www.partitionwizard.com/partitionmagic/uefi-vs-bios.html)
2. [Unified_Extensible_Firmware_Interface](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)