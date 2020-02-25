---
title: ubuntu清理磁盘空间
date: 2020-02-25 21:10:10
tags:
- ubuntu
categories:
- ubuntu
---

## 问题
虽然linux不像windows那样需要碎片整理，随着时间的流逝，硬盘空间会
越来越小，有必要对硬件空间清理。以ubuntu为例说明如何清理硬盘空间。

## 查看硬盘占用

```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            934M     0  934M   0% /dev
tmpfs           193M  2.6M  191M   2% /run
/dev/sda1       8.8G  6.5G  1.9G  78% /
tmpfs           965M     0  965M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           965M     0  965M   0% /sys/fs/cgroup
tmpfs           193M     0  193M   0% /run/user/0
tmpfs           193M     0  193M   0% /run/user/1000
/dev/sdb1       127G  3.0G  125G   3% /share
```
这里根分区可用空间不足，需要对根分区下目录空间占用进行分析。

## 查看根分区目录空间占用
```
$ du -h --max-depth=1 /
23M	/opt
0	/dev
236M	/boot
8.0K	/snap
5.8M	/lib32
3.1G	/usr
0	/sys
156M	/root
16K	/lost+found
0	/proc
12K	/media
4.0K	/srv
6.5M	/libx32
4.0K	/lib64
4.0K	/mnt
12M	/etc
1.1G	/lib
2.8G	/share
704M	/var
2.9M	/run
18M	/bin
1.1G	/home
19M	/sbin
48K	/tmp
4.0K	/nextcloud
9.2G	/
```

随着时间的流逝，log目录会比较大，可以通过journalctl命令控制日志文件所占空
间的大小。
```
$ journalctl --vaccum-size=100M
```

# 清空安装的软件包备份
ubuntu把安装的软件包缓存在如下目录:
```
$ ls /var/cache/apt/archives/
lock partial
```

然后通过下面的命令删除缓存的安装包。
```
$ apt clean
```

# 总结
通过以上步骤，硬盘空间大了许多。
