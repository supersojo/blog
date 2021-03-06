---
title: 分析一下目前linux内核栈的实现
date: 2020-02-27 21:10:10
tags:
- 内核栈
- CONFIG_VMAP_STACK
- scatterlist
categories:
- kernel
---
 
## 问题
最近在看内核一个[patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/net/sunrpc/auth_gss/gss_krb5_seqnum.c?h=v5.6-rc3&id=e7afe6c1d486b516ed586dcc10b3e7e3e85a9c2b)的修改，这个patch的commit描述
如下:
> While trying to reproduce a reported kernel panic on arm64, I discovered
> that AUTH_GSS basically doesn't work at all with older enctypes on arm64
> systems with CONFIG_VMAP_STACK enabled.  It turns out there still a few
> places using stack memory with scatterlists, causing krb5_encrypt() and
> krb5_decrypt() to produce incorrect results (or a BUG if CONFIG_DEBUG_SG
> is enabled).
描述里边提到CONFIG_VMAP_STACK内核宏和scatterlist。下面对它们分析。

## CONFIG_VMAP_STACK

### CONFIG_VMAP_STACK的功能
```
root@ubuntu:/share/linux# find . -name Kconfig | xargs grep -rsn VMAP_STACK
./arch/Kconfig:821:config HAVE_ARCH_VMAP_STACK
./arch/Kconfig:841:config VMAP_STACK
./arch/Kconfig:844:     depends on HAVE_ARCH_VMAP_STACK
./arch/arm64/Kconfig:135:       select HAVE_ARCH_VMAP_STACK
./arch/s390/Kconfig:133:        select HAVE_ARCH_VMAP_STACK
./arch/s390/Kconfig:714:        depends on !VMAP_STACK
./arch/x86/Kconfig:152: select HAVE_ARCH_VMAP_STACK             if X86_64
```
从搜索结果看，这个功能是和架构相关的。
再来看下该配置宏的具体描述：
> config VMAP_STACK
>        default y
>        bool "Use a virtually-mapped stack"
>        depends on HAVE_ARCH_VMAP_STACK
>        depends on !KASAN || KASAN_VMALLOC
>        ---help---
>          Enable this if you want the use virtually-mapped kernel stacks
>          with guard pages.  This causes kernel stack overflows to be
>          caught immediately rather than causing difficult-to-diagnose
>          corruption.
>
>          To use this with KASAN, the architecture must support backing
>          virtual mappings with real shadow memory, and KASAN_VMALLOC must
>           be enabled.

如果内核开启CONFIG_VMAP_STACK，内核可以快速检测内核栈overflow异常，比之前在内核栈溢出
访问时出问题难以诊断，内核栈溢出肯定访问垃圾数据，其结果不可预测的，难以排查，在有个guardpage后
只要内核栈溢出访问，内核可以快速捕获到。

内核栈的申请通过alloc_thread_stack_node实现。
```
static unsigned long *alloc_thread_stack_node(struct task_struct *tsk, int node)
{
#ifdef CONFIG_VMAP_STACK
        void *stack;
        int i;

        for (i = 0; i < NR_CACHED_STACKS; i++) {
                struct vm_struct *s;

                s = this_cpu_xchg(cached_stacks[i], NULL);

                if (!s)
                        continue;

                /* Clear the KASAN shadow of the stack. */
                kasan_unpoison_shadow(s->addr, THREAD_SIZE);

                /* Clear stale pointers from reused stack. */
                memset(s->addr, 0, THREAD_SIZE);


                tsk->stack_vm_area = s;
                tsk->stack = s->addr;
                return s->addr;
        }

        /*
         * Allocated stacks are cached and later reused by new threads,
         * so memcg accounting is performed manually on assigning/releasing
         * stacks to tasks. Drop __GFP_ACCOUNT.
         */
        stack = __vmalloc_node_range(THREAD_SIZE, THREAD_ALIGN,
                                     VMALLOC_START, VMALLOC_END,
                                     THREADINFO_GFP & ~__GFP_ACCOUNT,
                                     PAGE_KERNEL,
                                     0, node, __builtin_return_address(0));
        /*
         * We can't call find_vm_area() in interrupt context, and
         * free_thread_stack() can be called in interrupt context,
         * so cache the vm_struct.
         */
        if (stack) {
                tsk->stack_vm_area = find_vm_area(stack);
                tsk->stack = stack;
        }
        return stack;

#else
        struct page *page = alloc_pages_node(node, THREADINFO_GFP,
                                             THREAD_SIZE_ORDER);

        if (likely(page)) {
                tsk->stack = page_address(page);
                return tsk->stack;
        }
        return NULL;
#endif
}
```

为了避免vmalloc的频繁调用，内核使用cached_stacks本地缓存数组，保存进程退出时的内核栈指针。在
创建进程时可以直接从cached_stacks本地缓存数组获取可用的内核栈。

如果cached_stacks本地缓存数组不可用，则通过vmalloc接口申请内核栈。vmalloc申请的空间并不保证是
物理连续的页。

vmalloc申请的空间在使用上要注意：
1. 不能用于DMA操作
> DMA硬件没有页面映射，需要物理上连续的空间。
2. 不能用于scatterlist
> 后面分析

在不启用CONFIG_VMAP_STACK时，内核使用alloc_pages_node申请内核栈空间。这个接口申请连续的物理页
作为内核栈。

### 内核栈大小
前面分析了内核栈的分配，那么内核栈占多少空间呢？
> THREAD_SIZE宏定义了内核栈的大小。

THREAD_SIZE是架构相关的，在x86上定义为2个page的大小。
```
#define THREAD_SIZE_ORDER       1
#define THREAD_SIZE             (PAGE_SIZE << THREAD_SIZE_ORDER)
```

在x86_64上定义为如下大小，开启KASAN，则分配8个page的大小，不开启KSASAN则定义为4个page的大小。
```
#ifdef CONFIG_KASAN
#define KASAN_STACK_ORDER 1
#else
#define KASAN_STACK_ORDER 0
#endif

#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

这里提高了KASAN功能，这个是什么功能呢？
> Kasan 是 Kernel Address Sanitizer 的缩写，它是一个动态检测内存错误的工具，主要功能是
> 检查内存越界访问和使用已释放的内存等问题。Kasan 集成在 Linux 内核中，随 Linux 内核
> 代码一起发布，并由内核社区维护和发展。

这里不对KASAN功能进行分析，更多的可以参考下面的文档。
KASAN扩展阅读
1. [KASAN内核文档](https://www.kernel.org/doc/html/v4.14/dev-tools/kasan.html)
2. [KASAN简单介绍](https://www.ibm.com/developerworks/cn/linux/1608_tengr_kasan/index.html)

### 内核栈的布局
内核栈布局和架构相关，x86配置如下
```
$ cat arch/x86/Kconfig
config X86
        def_bool y
		...
		select THREAD_INFO_IN_TASK
		...

$ cat include/linux/sched.h
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
        /*
         * For reasons of header soup (see current_thread_info()), this
         * must be the first element of task_struct.
         */
        struct thread_info              thread_info;
#endif

```
由上可见在x86上，内核栈就是单纯的内核栈，thread_info结构不在内核栈的低端地址上，而是移到了task_struct结构中，这样好处是
内核栈溢出时不会破坏thread_info结构，更加安全。

之前内核栈的布局是这样的，如下所示：
```
high address                low address 
|----------------------|<-------->|
                        thread_info
```

## scatterlist
scatterlist用来描述一内存段，其结构如下：
```
struct scatterlist {
        unsigned long   page_link;
        unsigned int    offset;
        unsigned int    length;
        dma_addr_t      dma_address;
#ifdef CONFIG_NEED_SG_DMA_LENGTH
        unsigned int    dma_length;
#endif
};
```
page_link是page指针，这里复用该page指针使其支持[scatterlist chain](https://lwn.net/Articles/234617/)。
每个scatterlist描述一个物理页的内存片段。

为何vmalloc空间不能用于scatterlist呢？
```
          vmalloc buffer
           |-------------|
	|-------------|-----------|
         pageX	       pageY
```
如果vmalloc buffer跨page边界，看会发生什么。
```
$ cat net/sunrpc/auth_gss/gss_krb5_crypto.c
u32
krb5_encrypt(
        struct crypto_sync_skcipher *tfm,
        void * iv,
        void * in,
        void * out,
        int length)
{
        u32 ret = -EINVAL;
        struct scatterlist sg[1];
		...
		memcpy(out, in, length);
        sg_init_one(sg, out, length);
		...
}
$ cat lib/scatterlist.c
void sg_init_one(struct scatterlist *sg, const void *buf, unsigned int buflen)
{
        sg_init_table(sg, 1);
        sg_set_buf(sg, buf, buflen);
}
$ cat include/linux/scatterlist.h
static inline void sg_set_buf(struct scatterlist *sg, const void *buf,
			      unsigned int buflen)
{
#ifdef CONFIG_DEBUG_SG
	BUG_ON(!virt_addr_valid(buf));
#endif
	sg_set_page(sg, virt_to_page(buf), buflen, offset_in_page(buf));
}
$ cat include/linux/scatterlist.h
static inline void sg_set_page(struct scatterlist *sg, struct page *page,
			       unsigned int len, unsigned int offset)
{
	sg_assign_page(sg, page);
	sg->offset = offset;
	sg->length = len;
}
```
在sg_set_page后，scatterlist只记录了pageX，后面的pageY是没有记录的，后面使用scatterlist时
可能访问到垃圾数据。可见vmalloc空间时不能用于scatterlist的。

## 总结
本文分析了CONFIG_VMAP_STACK和scatterlist的功能，在使能CONFIG_VMAP_STACK后，内核栈上分配的
空间是物理上不连续的，不能用于scatterlist，不能用于DMA。