---
title: arm64 armel armhf区别
date: 2020-11-20 22:10:10
tags:
- arm
categories:
- arm
---

## 问题
ARM嵌入式开发，必定涉及到交叉编译工具链问题，下载交叉编译
工具链时总会有多个选择，我们该选择哪个工具链呢?

我们需要区分armel armhf arm64。

1. armel
ARM EABI(armel)支持老的32位arm。

2. armhf
较新的arm hard-float(armhf)支持新一点的32位arm，FPU没有标准化。

3. arm64
64位的arm支最新的64位arm，默认都带FPU了，已经标准化了。

## 背景知识
为何会有如此多种类的工具链需要选择呢？这有arm的历史背景有关。

其实最开始的只有一种arm，后来由于诸多原因如端模式到浮点的支持，它已经被被废弃了，后来的arm EABI (armel)取代了它。

armel（arm eabi little endian）仅仅是个名字而已，用于和老的版本做区分，可以认为它仅仅是对arm最基本的支持，可以被任何硬件运行。

armhf支持硬件FPU，计算更快，而armel则需要对浮点数模拟计算。


arm64默认就支持FPU。

gcc编译的时候，使用-mfloat-abi选项来指定浮点运算使用的
是哪种，soft不使用fpu，armel使用fpu，使用普通寄存器，armhf
使用fpu，使用fpu的寄存器。

