---
title: about-dwarf
date: 2020-01-20 22:33:52
tags:
---
# 关于dwarf
一种调试文件格式，用于支持源码级调试功能。它通过树结构描述一个程序，每个节点有子节点或兄弟节点。节点可以表示**类型**、**变量**、或**函数**。
dwarf使用一系列的DIE(debugging information entries)表示源程序。每个**DIE**包含一个tag标签标识和一系列属性。每个或多个DIE组成一组标识程序中某个实体，或是程序或者变量等等。
# readelf
readelf --debug-dump=info vmlinux
>输出.debug_info段中调试信息

下面举个例子。
`readelf --debug-dump=info vmlinux | grep dentry_open`

通过上面的命令可以得到如下信息
```
89773762- <1><aa9c983>: Abbrev Number: 57 (DW_TAG_subprogram)
89773763-    <aa9c984>   DW_AT_external    : 1
89773764-    <aa9c984>   DW_AT_declaration : 1
89773765:    <aa9c984>   DW_AT_linkage_name: (indirect string, offset: 0x105b80): dentry_open
89773766:    <aa9c988>   DW_AT_name        : (indirect string, offset: 0x105b80): dentry_open
89773767-    <aa9c98c>   DW_AT_decl_file   : 55
89773768-    <aa9c98d>   DW_AT_decl_line   : 2493
89773769-    <aa9c98f>   DW_AT_decl_column : 22
```
dentry_open是一个函数，在2493行被声明的。
可以用readelf查看各种信息。
在后面的参考中，[2]有对dwarf的详细介绍。

# 参考
1. [Exploring the DWARF debug format information](https://developer.ibm.com/technologies/systems/articles/au-dwarf-debug-format/)
2. [Debugging Tools Intro DWARF,ELF,GDB,build-id](http://people.redhat.com/jkratoch/DeveloperConference2011-debug.pdf)


