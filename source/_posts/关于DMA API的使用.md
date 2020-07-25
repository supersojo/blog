---
title: 关于如何使用DMA API
date: 2020-7-25 13:01:20
tags:
- dma
categories:
- kernel
---

# 关于dma api的使用

在[关于DMA API的内核文档](https://www.kernel.org/doc/Documentation/DMA-API-HOWTO.txt)中提到

```
Note that the DMA API works with any bus independent of the underlying
microprocessor architecture. You should use the DMA API rather than the
bus-specific DMA API, i.e., use the dma_map_*() interfaces rather than the
pci_map_*() interfaces.
```

推荐使用dma_map_\*() 而不是 pci_map_\*()。
dma api是独立于总线的接口，使用pci_map_\*()是不合适的。

使用该接口时包含相关的头文件，地址使用dma_addr_t。
```
#include <linux/dma-mapping.h>
dma_addr_t addr;
```

# 哪些内存可以用于DMA

能够用于DMA的内存在大部分平台上是要求物理连续的，因此下面的内存可以用于DMA。

1 page allocator or kmalloc or kmem_cache_alloc

可以直接把这些地址用于dma_map_\*()等api

2 vmalloc/vmap地址不可用于DMA，物理上不连续



# dma_set_mask_and_coherent

告诉内核外设对dma的支持情况，不管流式还是一致性dma都要这个步骤

如果有特别的需求，可以使用以下的只设置对应方式的dma

1 dma_set_mask 流式
2 dms_set_coherent_mask 一致性

如果以上设置失败，有两种方式
1 使用非dma方式
2 忽略外设


# 一致性dma

在驱动初始化时映射好，驱动卸载时接触映射，在外设工作中保证访问的一致性
常用于网卡的环形buffer，scsi的mailbox命令结构，设备固件等。

用于DMA的内存存在于驱动的整个生命周期中。

一致性dma不可避免的使用内存屏障。cpu可能执行指令重排，导致外设首先看到的不是想要的数据。

采用如下的内存屏蔽保证驱动正常工作。
```
desc->word0=address;
wmb();
desc->word1=DESC_VALID;
```

在某些平台上，驱动刷新写缓存，如写某个寄存器后再都寄存器，需要刷缓存。


# 流式dma

流式dma是指在dma传输时才映射，一旦传输完就解除映射。除非使用dma_sync_\*调用。


流式dma可以认为是异步dma

常用例子是外设的发送接收buffer，scsi外设读写的buffer。

注意dma地址是共享资源，流式dma会比较节省。


# 参考

1 [DMA API](https://www.kernel.org/doc/Documentation/DMA-API-HOWTO.txt)
