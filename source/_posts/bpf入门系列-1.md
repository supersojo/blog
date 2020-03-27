---
title: bpf入门系列-1
date: 2020-02-13 21:21:10
tags:
- bpf
categories:
- kernel
---
## bpf的build环境

简单讲，bpf是内核内置的虚拟机，对bpf的支持，内核需要开启编译选项如下
```
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_NET_CLS_BPF=m
CONFIG_NET_ACT_BPF=m
CONFIG_BPF_JIT=y
CONFIG_HAVE_BPF_JIT=y
CONFIG_BPF_EVENTS=y
```
[1]中是在oracle linux上面的介绍，在ubuntu1804上遇到很多问题，这里把问题列举下。

## build environment
```
Distro: ubuntu 1804
Kernel: 5.3.0-42
Packages:
linux-image-5.3.0-42-generic
linux-image-5.3.0-42-generic-dbgsym
linux-headers-5.3.0-42-generic
linux-headers-5.3.0-42
linux-source-5.3.0
llvm-9
clang-9
libelf-dev
```
在ubuntu1804上我安装了5.3.0的内核，llvm和clang都是安装的9的，另外要安装elf开发库。由于Makefile中使用clang和llc命令，这里建立软链接
```
ln -s /usr/bin/llc-9 /usr/bin/llc
ln -s /usr/bin/clang-9 /usr/bin/clang
```

把代码clone到本地
```
git clone https://github.com/oracle/linux-blog-sample-code.git
```

master分支只有README，源码文件在bpf-test分支上，所以切换分支
```
git checkout -b bpf-test origin/bpf-test
```

这时编译例子程序，有如下错误
```
$ cd bpf-test
$ make
...
LLVM ERROR: 'helper_test_init' label emitted multiple times to assembly file
Makefile:69: recipe for target 'test_bpf_helper_init_kern.o' failed
...
```
[这里](https://patchwork.ozlabs.org/patch/808209/)提到如果section名字和函数名字一样，clang会报错。我们可以
修改一下函数名字。
```
bpf/test_bpf_helper_init_kern.c
helper_test_init => helper_test_init_prog
```

继续编译发现还有类似错误，可以按照这种方式修改。由于其它地方使用了宏方式定义section，这里修改宏定义，在函数名后追加_prog
```
bpf/test_bpf_helper_kern.h

#define BPF_HELPER_TEST_FUNC(helper, name, direction)                   \
SEC(BPF_HELPER_TEST_NAME(helper, name, direction))                      \
static __always_inline int helper##_##name##_##direction##_prog(struct __sk_buff *skb)
```

这样bpf程序可以正确编译出来，后面开始运行例子程序。

## run problem
在运行程序时有如下错误
```
root@ubuntu:~/bpftest/linux-blog-sample-code/bpf-test# make test
make  -C bpf test
make[1]: Entering directory '/root/bpftest/linux-blog-sample-code/bpf-test/bpf'
make[1]: 'test' is up to date.
make[1]: Leaving directory '/root/bpftest/linux-blog-sample-code/bpf-test/bpf'
make  -C user test
make[1]: Entering directory '/root/bpftest/linux-blog-sample-code/bpf-test/user'
bash test_bpf_helper_run.sh
before setup
0 maps not supported in current map section!
Error fixing up map structure, incompatible struct bpf_elf_map used?
Error fetching ELF ancillary data!
Unable to load program
setup ok
could not find map bpf_helper_test_map: No such file or directory
Makefile:63: recipe for target 'test' failed
make[1]: *** [test] Error 1
make[1]: Leaving directory '/root/bpftest/linux-blog-sample-code/bpf-test/user'
Makefile:69: recipe for target 'test-user' failed
make: *** [test-user] Error 2
```

大致是说在bpf程序中找不到map。

测试命令执行的是以下命令
```
tc filter add dev veth1 ingress bpf da \
            obj ../bpf/test_bpf_helper_init_kern.o \
            sec helper_test_init
```
tc是iproute2包中的命令，这里是iproute2的[upstream](https://github.com/shemminger/iproute2)地址。

```
git clone https://github.com/shemminger/iproute2.git
grep "maps not supported" -rsn ./
...
./lib/bpf.c:1886:               fprintf(stderr, "struct bpf_elf_map too small, not supported!\n");
...
```
看下报错的代码
```
static int bpf_fetch_maps_end(struct bpf_elf_ctx *ctx)
{
        struct bpf_elf_map fixup[ARRAY_SIZE(ctx->maps)] = {};
        int i, sym_num = bpf_map_num_sym(ctx);
        __u8 *buff;

        if (sym_num == 0 || sym_num > ARRAY_SIZE(ctx->maps)) {
                fprintf(stderr, "%u maps not supported in current map section!\n",
                        sym_num);
...
static int bpf_map_num_sym(struct bpf_elf_ctx *ctx)
{
	int i, num = 0;
	GElf_Sym sym;

	for (i = 0; i < ctx->sym_num; i++) {
		int type;

		if (gelf_getsym(ctx->sym_tab, i, &sym) != &sym)
			continue;

		type = GELF_ST_TYPE(sym.st_info);
		if (GELF_ST_BIND(sym.st_info) != STB_GLOBAL ||
		    (type != STT_NOTYPE && type != STT_OBJECT) ||
		    sym.st_shndx != ctx->sec_maps)
			continue;
		num++;
	}

	return num;
}

```
可见bpf_map_num_sym返回了0。以上是iproute2的upstream代码，检查符号类型，如果不是STT_NOTYPE和STT_OBJECT则跳过，我们需要看下ubuntu中iproute2的代码。
安装ubuntu中iproute2源码包。

下面命令会安装ubuntu仓库中最新的iproute2源码包
```
$ apt source iproute2
```

可能你需要特定版本的源码包，这时需要检查系统iproute2的版本号。
```
root@ubuntu:~/iproute2/debian# dpkg -l | grep iproute2
ii  iproute2                                     4.15.0-2ubuntu1                                 amd64        networking and traffic control tools
root@ubuntu:~# apt source iproute2=4.15.0
```

这样我们可以检查对应版本4.15.0的iproute2的中bpf_map_num_sym的实现。
```
static int bpf_map_num_sym(struct bpf_elf_ctx *ctx)
{
        int i, num = 0;
        GElf_Sym sym;

        for (i = 0; i < ctx->sym_num; i++) {
                if (gelf_getsym(ctx->sym_tab, i, &sym) != &sym)
                        continue;

                if (GELF_ST_BIND(sym.st_info) != STB_GLOBAL ||
                    GELF_ST_TYPE(sym.st_info) != STT_NOTYPE ||
                    sym.st_shndx != ctx->sec_maps)
                        continue;
                num++;
        }

        return num;
}
```
检查符号的类型，如果不是STT_NOTYPE则跳过。这里和upstream实现是有区别的，我们接下来需要确认我们生成的bpf程序中符号的类型。


首先确认有maps段
```
root@ubuntu:~/bpftest/linux-blog-sample-code/bpf-test# llvm-readelf-9 --sections bpf/test_bpf_helper_init_kern.o

There are 23 section headers, starting at offset 0x3068:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .strtab           STRTAB          0000000000000000 002e10 000256 00      0   0  1
  [ 2] .text             PROGBITS        0000000000000000 000040 000000 00  AX  0   0  4
  [ 3] helper_test_init  PROGBITS        0000000000000000 000040 000010 00  AX  0   0  8
  [ 4] .rodata.str1.1    PROGBITS        0000000000000000 000050 00025f 01 AMS  0   0  1
  [ 5] .data             PROGBITS        0000000000000000 0002b0 0001c0 00  WA  0   0 16
  [ 6] .rel.data         REL             0000000000000000 0023c0 000370 10     22   5  8
  [ 7] maps              PROGBITS        0000000000000000 000470 00001c 00  WA  0   0  4
  [ 8] license           PROGBITS        0000000000000000 00048c 000004 00  WA  0   0  1
  [ 9] .debug_str        PROGBITS        0000000000000000 000490 0003c3 01  MS  0   0  1
  [10] .debug_abbrev     PROGBITS        0000000000000000 000853 000103 00      0   0  1
  [11] .debug_info       PROGBITS        0000000000000000 000956 000568 00      0   0  1
  [12] .rel.debug_info   REL             0000000000000000 002730 000680 10     22  11  8
  [13] .debug_macinfo    PROGBITS        0000000000000000 000ebe 000001 00      0   0  1
  [14] .BTF              PROGBITS        0000000000000000 000ebf 0006b9 00      0   0  1
  [15] .rel.BTF          REL             0000000000000000 002db0 000020 10     22  14  8
  [16] .BTF.ext          PROGBITS        0000000000000000 001578 000058 00      0   0  1
  [17] .rel.BTF.ext      REL             0000000000000000 002dd0 000020 10     22  16  8
  [18] .eh_frame         PROGBITS        0000000000000000 0015d0 000030 00   A  0   0  8
  [19] .rel.eh_frame     REL             0000000000000000 002df0 000010 10     22  18  8
  [20] .debug_line       PROGBITS        0000000000000000 001600 000141 00      0   0  1
  [21] .rel.debug_line   REL             0000000000000000 002e00 000010 10     22  20  8
  [22] .symtab           SYMTAB          0000000000000000 001748 000c78 18      1 129  8
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

从上面的输出看有maps段，再看下maps段的符号
```
./bpf/test_bpf_helper_kern.h
struct bpf_elf_map SEC("maps") bpf_helper_test_map = {
        .type = BPF_MAP_TYPE_HASH,
        .size_key = sizeof(long),
        .size_value = sizeof(long),
        .pinning = PIN_GLOBAL_NS,
        .max_elem = 256,
};
root@ubuntu:~/bpftest/linux-blog-sample-code/bpf-test# llvm-readelf-9 -s bpf/test_bpf_helper_init_kern.o | grep bpf_helper_test_map
   130: 0000000000000000    28 OBJECT  GLOBAL DEFAULT    7 bpf_helper_test_map

```
哇，这里bpf_helper_test_map的类型是OBJECT，tc程序认为不是有效的符号而跳过了。我们需要修改iproute2的代码重新打包，注意有3个函数都需要修改。
```
bpf_map_num_sym
bpf_map_verify_all_offs
bpf_map_fetch_name
```

关于debian下编译打包参考[这里](https://moeclub.org/2017/03/31/100/?spm=38.1)
这里列下简单步骤
```
mkdir YourPackage
cd YourPackage
apt source -t bionic iproute2
apt build-dep -y -t bionic iproute2
vim lib/bpf.c  ==> modify the source code
dpkg-buildpackge --commit
dpkg-buildpackge 
dpkg -i ../iproute2*.deb
```

这时例子程序可以成功运行了。
```
make test
```

## reference

1. [How to build bpf program out of the kernel tree](https://blogs.oracle.com/linux/notes-on-bpf-4)