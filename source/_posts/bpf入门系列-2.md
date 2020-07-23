---
title: bpf入门系列-2
date: 2020-02-17 21:21:10
tags:
- bpf
categories:
- kernel
---

## 头文件设置

要想能够成功编译bpf程序，需要正确的设置头文件，在linux内核版本3.7之后引入了uapi，下面是uapi的简单介绍。

首先介绍下[uapi](https://stackoverflow.com/questions/18858190/whats-in-include-uapi-of-kernel-source-project)。
uapi主要是以下两个目录
```
kernel_dir/include/uapi
kernel_dir/arch/$ARCH/include/uapi
```

uapi的目的是让头文件更加清晰，只有上面的头文件是用户态可以使用的，其它的作为内核私有头文件使用。

在编译bpf程序时我们要用到内核头文件，而不仅仅上上面的uapi的头文件
```
kernel_dir/arch/$ARCH/include
kernel_dir/arch/$ARCH/include/generated
kernel_dir/include
```
关于[这个例子](https://blog.linote.cn/2020/03/27/bpf%E5%85%A5%E9%97%A8%E7%B3%BB%E5%88%97-1/)的编译，如果使用内核头文件会遇到问题，例子中带了include/types.h，这个头文件和内核头文件冲突，可行workaround方案是把自带include放到编译选项的最后。
**编译器在寻找头文件是总是按照编译选项中列出目录先后的顺序搜索**。

另外看下uapi/linux/types.h文件
```
#ifndef _UAPI_LINUX_TYPES_H
#define _UAPI_LINUX_TYPES_H

#include <asm/types.h>

#ifndef __ASSEMBLY__
#ifndef __KERNEL__
#ifndef __EXPORTED_HEADERS__
#warning "Attempt to use kernel headers from user space, see http://kernelnewbies.org/KernelHeaders"
#endif /* __EXPORTED_HEADERS__ */
#endif

#include <linux/posix_types.h>

```
我们在使用时需要定义__KERNEL__宏。

一个可用的头文件包含选项
```
linuxhdrs ?= /usr/src/linux-headers-`uname -r`
LINUXINCLUDE =  -I$(linuxhdrs)/arch/x86/include \
                -I$(linuxhdrs)/arch/x86/include/generated \
                -I$(linuxhdrs)/arch/x86/include/uapi \
                -I$(linuxhdrs)/arch/x86/include/generated/uapi \
                -I$(linuxhdrs)/include/generated/uapi \
                -I$(linuxhdrs)/include \
                -I$(linuxhdrs)/include/uapi \
                -include $(linuxhdrs)/include/linux/kconfig.h

```

在头文件设置好后，还需要额外编译选项用于编译bpf程序
```
CLANG ?= clang
INC_FLAGS = -nostdinc -isystem $(shell $(CLANG) -print-file-name=include)
EXTRA_CFLAGS ?= -O2 -g -Wall -emit-llvm
LLC ?= llc
CLANG ?= clang
LLVM_OBJDUMP ?= llvm-objdump

$(KERNELOBJS):  %.o:%.c
        $(CLANG) $(INC_FLAGS) \
                -D__KERNEL__ -D__ASM_SYSREG_H \
                $(DEBUG_FLAGS) \
                -Wno-unused-value -Wno-pointer-sign \
                -Wno-compare-distinct-pointer-types \
                -Wno-gnu-variable-sized-type-not-at-end \
                -Wno-address-of-packed-member -Wno-tautological-compare \
                -Wno-unknown-warning-option \
                $(LINUXINCLUDE) \
                -I../include \
                $(EXTRA_CFLAGS) -c $< -o -| $(LLC) -march=bpf -filetype=obj -o $@
```

## 一个bpf程序框架

这里是我整理的一个bpf框架，地址是[bpf_skeleton](https://github.com/supersojo/bpf_skeleton)

这里讲下遇到的问题。首先是内核部分，按照上面一节设置头文件后问题不大，

```
root@ubuntu:~/bpftest/bpf_skeleton# ls kern
bitehist.c  bpf_endian.h  bpf_helpers.h  Makefile
```

可能对为何需要bpf_helpers.h有疑问，bpf程序最终是编译成bpf字节码，由bpf虚拟机执行，这里把bpf函数设置为整数标识，最后由llvm生成对bpf内核函数的调用。

```
static void *(*bpf_map_lookup_elem)(void *map, void *key) =
        (void *) BPF_FUNC_map_lookup_elem;
static int (*bpf_map_update_elem)(void *map, void *key, void *value,
                                  unsigned long long flags) =
        (void *) BPF_FUNC_map_update_elem;
```
上面只是对bpf函数赋值为bpf内核函数对应的整数标识

为使用libbpf，这里通过在内核源码目录下编译安装
```
cd kernel_dir/tools/lib/bpf
make && make install && make install_headers
```
这样缺省头文件安装到/usr/local/include，库文件安装到/usr/local/lib64下面

```
root@ubuntu:~/bpftest/bpf_skeleton# ls user/
bitehist.c       bpf_load.c  linux     perf-sys.h
bitehist_kern.o  bpf_load.h  Makefile
```
可能注意到为何有linux/types.h，uapi不是有对应头文件吗? 使用uapi下的types.h需要定义__KERNEL__，这里使用tools下自带的，可以不依赖内核头文件就可以编译用户态代码
bpf_load.c是从samples/bpf下拷贝过来的，里面是bpd加载的实用函数，非常方便。

## 总结

经过上面步骤，我们拥有了一个可以使用的bpf框架，开始编写自己的bpf程序吧

## 参考
1. [BPF tools](https://github.com/brendangregg/BPF-tools/tree/master/new/2019-05-21)
