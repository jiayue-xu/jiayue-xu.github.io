---
title: DMA直接内存访问
date: 2024-02-24 14:09:10
---

DMA是一种硬件机制，用于支持在**外设**与**主机内存**之间直接进行数据读写，而无需CPU的参与。由于CPU全程没有参与，因此硬件需要通过中断信号通知CPU DMA的完成状态。当然CPU也可以选择轮询寄存器的方式获取DMA的完成状态，但这就有点两边都不讨好了，会让CPU和DMA控制器都处于繁忙状态，在重数据轻计算场景下可能会看到这种用法。

<!--more-->

## DMA传输使用什么地址？

外设和主机内存都在系统总线上，对于不同的系统总线，DMA传输使用的地址类型也不一样，对于PCIe总线而言，如果没有使用IOMMU机制，DMA使用的是物理地址，对于SBus而言，使用的虚拟地址。如果DMA传输支持虚拟地址，单次DMA传输大于一页时，不要求物理页地址跨页连续。

不是所有内存申请接口分配出来的内存buffer都适合用来进行DMA传输的，比如：如果分配出来的Buffer对应的内存物理地址（总线地址）高于4GB，而设备的DMA寻址能力只有32-bit，那这块Buffer就不能被这个设备用于DMA传输。

在使用kmalloc和get_free_pages时，如果指定了`GFP_DMA`，申请的内存就在24-bit范围内。但是要注意，申请的内存大小不是无限制的，比如get_free_pages，MAX_ORDER为11，至多申请4MB。

当你需要传输很大的DMA Buffer时，有以下几种方式：

1. 在系统boot阶段使用`mem=`参数指定保留部分高地址空间用于DMA传输
2. 如果设备支持的话，使用scatter/gather IO方式进行传输

不同的架构有不同的DMA一致性限制，有些很容易就能实现一致性，有些很麻烦，Linux提供了DMA抽象层屏蔽了这些问题，常见的接口如下：

## 指定设备寻址能力

```c
#include <linux/dma-mapping.h>

int dma_set_mask(struct device *dev, u64 mask);
```

## DMA映射

DMA映射做的事情是返回一个设备可用的总线地址，看起来virt_to_bus函数也许能做这件事，但实际上在特殊场景下会遇到一些问题，比如：

- 如果系统开启了IOMMU，需要把IOVA给硬件，硬件使用IOVA进行DMA传输
- 如果应用程序malloc了一块位于4GB以上的内存空间，但是硬件的寻址能力为32-bit，此时就需要使用**bounce buffer**，牺牲一点DMA的性能，保证DMA可以正常进行。

因此无法直接调用virt_to_bus把对应的总线地址给硬件，最好使用DMA通用接口进行DMA映射。

DMA映射做的另一件事情是保证缓存一致性，在一些架构上通过硬件就可以实现一致性，在一些架构上则需要软件配合。

有两种DMA映射方式：一致DMA映射、流式DMA映射。

### 一致DMA映射

```c
void *dma_alloc_coherent(struct device *dev, size_t size, dma_addr_t *dma_handle, int flag);

void dma_free_coherent(struct device *dev, size_t size, void *vaddr, dma_addr_t dma_handle);
```

此外，DMA pool用于小块的一致性DMA映射，dma_alloc_coherent至少会分配一页大小的DMA Buffer：

```c
struct dma_pool* dma_pool_create(const char *name, struct device *dev, size_t size, size_t align, size_t allocation);

void dma_pool_destroy(struct dma_pool *pool);

void *dma_pool_alloc(struct dma_pool *pool, int mem_flags, dma_addr_t *handle);

void dma_pool_free(struct dma_pool *pool, void *vaddr, dma_addr_t addr);
```

### 流式DMA映射

流式DMA映射必须指定数据传输方向，可以用于简化缓存一致性操作：

```c
enum dma_data_direction {
    DMA_TO_DEVICE,
    DMA_FROM_DEVICE,
    DMA_BIDIRECTIONAL,
    DMA_NONE,
};
```

针对不同的场景，使用不同的接口进行DMA映射：

```c
dma_addr_t dma_map_single(struct device *dev, void *buffer, size_t size, enum dma_data_direction);

void dma_unmap_single(struct device *dev, dma_addr_t dma_addr, size_t size, enum dma_data_direction);

dma_addr_t dma_map_page(struct device *dev, struct page *page, unsigned long offsetm, size_t size, enum dma_data_direction);

void dma_unmap_page(struct device *dev, dma_addr_t dma_address, size_t size, enum dma_data_direction direction);

int dma_map_sg(struct device *dev, struct scatterlist *sg, int nents, enum dma_data_direction direction);

void dma_unmap_sg(struct device *dev, struct scatterlist *list, int nents, enum dma_data_direction direction);

void dma_sync_sg_for_cpu(struct device *dev, struct scatterlist *sg, int nents, enum dma_data_direction direction);

void dma_sync_sg_for_device(struct device *dev, struct scatterlist *sg, int nents, enum dma_data_direction direction);
```

当dma_map_single被调用时，对应的数据可能还在缓存中，该函数会将其flush到内存；当数据地址无法被设备寻址时，dma_map_single会创建一个bounce buffer，如果DMA方向为双向的，在map、unmap时都需要拷贝一次数据，这是比较耗时的。

如果不想unmap又想访问DMA Buffer，需要调用：

```c
void dma_sync_single_for_cpu(struct device *dev, dma_addr_t bus_addr, size_t size, enum dma_data_direction);

void dma_sync_single_for_device(struct device *dev, dma_addr_t bus_addr, size_t size, enum dma_data_direction);
```
