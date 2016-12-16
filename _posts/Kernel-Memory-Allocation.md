---
published: true
date: 2007-04-08T13:00:00.000Z
tags:
  - Linux
  - Kernel
  - Memory
  - Cache
  - buddy

title: Kernel中的内存分配方式
---
这周BSP那边碰到一个蛮严重的issue： 循环放电影，v4l2 output driver的 dma_alloc很容易就失败，kernel panic，dump出当前buddy系统的状态。
初步分析是由于内存fragment导致没有足够大的连续内存分配给v4l output driver。开始debug
首先通过/proc/buddyinfo 在播放电影时候不间断的dump buddy状态，发现大块内存块：256K － 4MB block 减少的很迅速，1－2小时后，buddy中就不存在大块连续内存。这证明了初步分析是正确的，但是还不知道是谁导致了fragment，是应用层，还是v4l2 driver本身？
接下来添加一个NORMAL zone（ARM体系中就DMA一个zone，所以application和dma都使用一个buddy系统），让普通内存分配和dma隔开。继续测试，发现dma zone中的内存块状态在最初很正常，但NORMAL zone的内存却在急剧减少，最终NORMAL zone已经没有足够内存给application，kernel转而向DMA zone申请，最终issue重现。

现在问题很清楚了，application发生内存leak，很严重的leak。但是dma_alloc这边会没有问题马？Fred在看了我描述的问题后，指出了dma_alloc中算法的一些问题：当向buddy申请完足够size的2^的内存块后，该函数会释放在申请块中多余的page，加速了fragment。后来我们做了一个实验，将dma_alloc里的释放多余页的功能去掉，fragment的速度大大减缓。
经过上周对这个issue的研究，我开始重新总结kernel里对内存分配的方式和方法，总结如下：

<!-- more -->

## 页分配 ##

**unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order)**

直接从buddy系统中获得原始页。最原始的分配方式。

## slab分配器 ##

1.通用 cache

**void kmalloc(size_t size, gfp_t flags)**

kmalloc 基于以下几种size的mem cache：32, 64, 128, 256, 512, 1,024, 2,048, 4,096, 
8,192, 16,384, 32,768, 65,536 和 131,072 bytes。其本质也是调用kmem_cache_alloc来分配
object。所以kmalloc一次最大可分配的size为128KB。kmalloc分配速度很快，在分配时需注意gfp flag
参数：在不interrupt上下文（ISR, softirq, tasklet）及不可睡眠上下文使用GFP_ATOMIC。
内核还增加了内存清零的分配函数：kzalloc。

2.专用 cache

**kmem_cache_create()**
**void kmem_cache_alloc(struct kmem_cache cachep, gfp_t flags)**

如果你需要频繁的分配和释放某个结构，建议不要采用kmalloc，而是自己在slab系统中创建memory cache。
指定该结构的object size。分配时使用kmem_cache_alloc。同样的slab object大小也有限制，一般情况
下一个MAX_OBJ_ORDER是5，也就是32个页，128KB。

## 非连续内存分配 ##

**void vmalloc(unsigned long size)**

超过128KB的内存显然不能使用slab分配，并且当申请的连续内存大小不能在buddy系统中得到满足，那么
就需要使用vmalloc。vmalloc为了把物理的非连续页一个个映射，从而导致比直接内存映射大的多的
后援缓冲区抖动。除非需要特别大的内存，否则尽量不要使用vmalloc。

## 基于DMA 分配 ##

**void  dma_alloc_coherent(struct device dev, size_t size, dma_addr_t handle, gfp_t gfp)**

在某些arch中，可以使用dma_alloc_coherent来分配DMA专用内存。列入在arch/arm/mm/consistent.c
中，该函数先分配最小可满足size的2^order内存，然后释放2^order-size多余的页给buddy。而arch/i386/
kernel/pci-dma.c中，则直接分配2^order块内存。

## 直接映射分配 ##

**ioremap(unsigned long phys_addr, size_t size)**
**int remap_pfn_range(struct vm_area_struct vma, unsigned long addr,**
**                    unsigned long pfn, unsigned long size, pgprot_t prot)**

在某些体系结构中，我们可以保留memory map段上的某一个区域，作为dma或其他设备的专有内存。
这段内存并不在kernel buddy的控制之下（没有被放入mem_maps），你也无法从以上几种分配方式中得到
这些内存。这个时候，你可以用ioremap和remap_pfn_range将这段内存直接映射到vm上。
