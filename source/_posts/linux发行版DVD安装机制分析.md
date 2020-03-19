---
title: linux发行版DVD安装机制分析
date: 2017-04-12 22:10:10
tags:
- CD
- DVD
- initrd
- vmlinuz
- squashfs
- ISO 9660
categories:
- linux
---
## linux发行版DVD安装机制分析

想知道linux发行版究竟是怎么启动安装的吗，下面简单的分析下dvd光盘启动linux机制。

## ISO 9660 标准

如果系统支持DVD启动，CD/DVD镜像格式必须满足标准，目前主要是采用[ISO 9660标准](http://www.ecma-international.org/publications/files/ECMA-ST/Ecma-119.pdf)。

ISO 9660是CD-ROM的标准文件系统。它还广泛用于DVD和BD介质上，也可能存在于USB sticks或硬盘上。

ISO 9660基本信息：

1. 扇区大小
   ISO 9660扇区正常都是2KB，尽管规范说可选，但是很难找到不是2K扇区大小的DVD
2. 数字编码
   支持大段和小端模式   
3. 字符串编码
   采用ASCII编码，但并不是所有字符都可用。但并不是所有DVD遵循这个规范。
4. 大小限制
   最大2^32的block，即8TB。
5. 系统区域
   前32KB(16个扇区)可以存储任意数据。如果ISO 9660文件系统存放在usb sticks或者硬盘，则一般用于存放boot代码(MBR)。
   
## 启动过程

BIOS/EFI固件支持CD/DVD启动，根据ISO 9660标准寻找光盘上的boot record记录，根据boot record进而找到boot image(isolinux.bin)，
执行bootimage代码，根据isolinux.cfg的配置加载内核(vmlinuz)和RAM DISK(initrd)，加载完后跳转到
内核执行。

这个initrd在内核启动时作为内存根文件系统使用，加载必要的驱动，然后内核切换到真正的根文件系统上面。
对于RHEL系就是光盘LiveOS下的squashfs.img。内核会挂载squashfs镜像，然后切换到其中的根文件系统。
```
[root@vm]# file squashfs.img
squashfs.img: Squashfs filesystem, little endian, version 4.0, 433674553 bytes, 3 inodes, blocksize: 131072 bytes, created: Wed Mar 18 13:21:34 2020
```
挂载该squashfs镜像后可得到里边是一个rootfs.img(ext4文件系统镜像)。
这个rootfs.img就是内核在安装发行版时使用的根文件系统。

### vmlinuz的生成

vmlinuz是内核文件，如果是自己编译内核，可以到[官网](https://www.kernel.org/)下载对应版本的
源码，编译。对于RHEL系发行版，一般有对应版本的源码包，如RPM源码包，安装后可以使用
[rpmbuild编译内核](https://fedoraproject.org/wiki/Building_a_custom_kernel/Source_RPM)。
对于debian系发行版，可以参考[这里](https://blog.lpxin.com/2019/05/15/%E4%BD%BF%E7%94%A8make-kpkg%E7%BC%96%E8%AF%91deb%E5%86%85%E6%A0%B8%E5%AE%89%E8%A3%85%E5%8C%85/)。

```
[root@vm]# file vmlinuz
vmlinuz: Linux kernel x86 boot executable bzImage, version 3.10.0-514.el7.x86_64 (root@RX300S8-1) #1 SMP Wed Mar 18 15:05:, RO-rootFS, swap_dev 0x5, Normal VGA
```

### initrd的解压和打包

```
[root@vm]# file initrd.img
initrd.img: LZMA compressed data, streamed
```

一般initrd都是经过压缩的，如上是采用lzma压缩的。需要lzma解压，完成后得到cpio包。
```
cpio -idmv <initrd.img
```

经过cpio命令可以解压得到里边的内容。
```
[root@vm]# ls
bin  etc   lib    proc  run   shutdown  sysroot  usr
dev  init  lib64  root  sbin  sys       tmp      var
```
这时可以对里边的内容修改，如替换成自己的内核驱动等等。

```
find . | cpio -c -o ../initrd.img
```
上面用于打包cpio，然后经过lzma压缩就可以得到可用的initrd。

这里注意如果使用自己的lib/modules/xxxx替换，需要depmod一下，生成
对应的modules.alias等文件，这样系统才能正常工作。

```
depmod -a -b <lib/modules/xxxx中lib的父目录>  -F <Your System.map文件>  -E <Your symvers文件> <lib/modules/xxxx中的xxxx字符串即内核版本号>
```
经过上述步骤后打包成的initrd是可以正常工作的。

### squashfs镜像的解压和打包

```
mkdir tmp_squashfs
mount squashfs.img tmp_squashfs
[root@vm tmp_squashfs]# tree
.
└── LiveOS
    └── rootfs.img

1 directory, 1 file
```
如果需要修改其中的rootfs.img需要拷贝出来，然后挂载该rootfs镜像，修改完直接卸载。
然后重新做成squashfs镜像。
```
mksquashfs <Your squashfs目录> squashfs.img
```

## 总结

经过上述步骤可以详细了解linux发行版启动安装机制。
简单讲首先运行一个linux内核(CD/DVD上面)，然后执行发行版的安装程序完成具体的安装。
安装程序负责对硬盘分区，挂载，使用CD/DVD上的目录作为安装源安装软件，最好配置重启完成安装。

