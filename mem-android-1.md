# Android内存占用分析(一)

> 分享者：李示刚   2019.12.13

&nbsp; 


本文以Android P为例，对应kernel版本为4.14


### 1、 MemTotal

MemTotal 即 /proc/meminfo 中的第一行的值, 可以认为是系统可供分配的内存总大小, 通常大小会比实际物理内存小, 这个是为什么呢? 少的部分被谁占用了呢?


#### 1.1 memblock

首先需要了解一下memblock.

在伙伴系统(buddy system)初始化完成前，Linux使用memblock来管理内存，memblock管理的内存分为两部分: `memory`类型和`reserved`类型。 对应的描述变量分别是`memblock.memory` 和 `memblock.reserved`。

memblock中两种类型的内存申请/添加函数如下:

````c
//memory 类型的memblock申请/添加
int __init_memblock memblock_add(phys_addr_t base, phys_addr_t size)
int __init_memblock memblock_add_node(phys_addr_t base, phys_addr_t size, int nid)

//reserved 类型的mblock申请/添加
int __init_memblock memblock_reserve(phys_addr_t base, phys_addr_t size)
````

##### 1.1.1 物理内存分布

memblock是如何知道物理内存的分布的呢?

kernel启动的过程中,从lk/uboot知道了DTB的加载地址, 在如下的调用流程中解析DTB下的memroy节点,将节点下的物理内存区间使用`memblock_add()`添加给memblock维护.
由于lk/uboot中可能使用内存,lk/uboot也可以修改DTB,所以这里memory节点下可能有多个物理内存区间. 多个物理区间中不连续的部分就是已经被lk/uboot占用的部分.

````
setup_machine_fdt(__fdt_pointer)   //__fdt_pointer是传递的DTB加载地址
  |-->early_init_dt_scan()
    |-->early_init_dt_scan_memory() //解析memory node,遍历其下的所有单元
      |-->early_init_dt_add_memory_arch()
        |-->memblock_add(base, size) //每个单元的起始地址和大小添加到memblock的memory类型中
````

````c
    //dts中的初始描述,0x40000000是起始物理地址
    //lk/uboot可以修改DTB中的节点,这里的reg单元可能有多个
    memory {
        device_type = "memory";
        reg = <0 0x40000000 0 0x20000000>;
    };   
````


可以通过 /sys/kernel/debug/memblock/ 下的节点查看两种内存的物理空间分布:

>lsg@eebbk:~$adb shell cat /sys/kernel/debug/memblock/memory
>lsg@eebbk:~$adb shell cat /sys/kernel/debug/memblock/reserved



memblock构建了内存的物理空间分布, 之后在伙伴系统(buddy system)初始化的过程中,构建了物理内存分布到虚拟内存空间的映射. 
memblock管理的`memory`类型的页框都添加到各ZONE中, `totalram_pages`统计了它的页框数目. 详细代码见 `free_all_bootmem()`

#### 1.2 隐藏的内存占用

这里说几个概念:

- 物理内存：    即DRAM物理内存大小,比如物理内存为2GRAM,则物理内存大小为2097152K
- memblock管理的内存:  memblock管理的内存, 包含伙伴系统管理的内存和reserved两部分,其页框数为 `get_num_physpages()`
  在下面的例子中, 可管理内存大小为 2045952K
- 伙伴系统管理的内存：伙伴系统管理的内存, 初始时对应memblock中的`memory`类型,其页框数在kernel中对应全局变量 `totalram_pages`
  在下面的例子中, 可分配内存大小为1983136K

**关系:**   
物理内存 > memblock管理的内存 > 伙伴系统管理的内存  
物理内存 = memblock管理的内存 + 预申请内存  
memblock管理的内存 = 伙伴系统管理的内存 + reserverd内存(memblock的`reserved type`)  

**预申请内存:**  
预申请内存是指在memblock初始化前已经申请的内存，呈现给memblock的是这部分物理内存不存在，比如lk/uboot/bootloader用到的内存。 
不同平台这部分的占用大小不尽相同，有的可能为0.  


**Q1:**
memblock是怎么知道有些内存已经被占用的?  
**A1:**
lk/uboot/bootloade修改DTB中memory节点下的空间区段(reg), kernel中读取这些空间区段(reg),空间区段之间的内存空间就是被预申请已经占用了的.  


````c
//kernel log 中的输出, 代码实现在 mem_init_print_info()
//关系:2097152K(物理2GB) = 2045952K(memblock管理的内存) + 51200K(预申请内存)
//关系:2045952K(memblock管理的内存) = 1982176K + 63776K + 0K
//1982176K 是此时 totalram_pages*4K 的大小, 也即memblcok管理的`memory type` 部分
//63776K 是memblcok管理的`reserved type` 部分
[    0.000000] -(0)[0:swapper]Memory: 1982176K/2045952K available (12924K kernel code, 1384K rwdata, 4392K rodata, 960K init, 5936K bss, 63776K reserved, 0K cma-reserved)

///proc/meminfo的输出
//1983136 是此时 totalram_pages*4K 的大小
lsg@eebbk:~$adb shell cat /proc/meminfo
MemTotal:        1983136 kB
````

**Q2:**
为什么MemTotal的大小比实际物理内存小?  
**A2:**
这个部分比实际的物理内存小,就是少了上面说的两个部分:预申请内存和kernel reserved内存.
预申请内存: 这部分的大小可以查看`/d/memblock/memory`相对物理内存空闲的部分. 
reserved内存: 这部分大小可以查看`/d/memblock/reserved`的大小,或者kernel log中的大小(上例中63776K)

**Q3:**
为什么开机时 `totalram_pages`的大小(上例中1982176K) 和 `totalram_pages`的大小(上例中1983136K) 开机后 不一致呢?  
**A3:**
这是因为memblock管理的`reserved type`中部分内存在初始化完毕后释放了,添加到了伙伴系统(buddy system)中. 详细代码见`free_initmem()` 
比如上面的log, 1982176K + 960K = 1983136K

><6>[    2.258901] -(2)[1:swapper/0]Freeing unused kernel memory: 960K

**Q4:**
kernel代码部分的占用在哪个部分体现?  
**A4:**
这部分包含 kernel code+rwdata+rodata+init+bss , 在 kernel log中已经输出. 这部分的物理占用计算在reserved部分.详细代码见 `arm64_memblock_init()`.
另外kernel code物理区域也会vmap到虚拟地址空间，其vmap的区间可以看 `/proc/vmallocinfo`中带有`paging_init`的行。


#### 1.3 reserved包含哪些

前面说到，memblock管理的内存, 包含伙伴系统管理的内存和reserved两部分。
那reserverd部分的内存占用又包含哪些呢？详细分解来看，至少包含以下这些部分：

1、代码
包含 kernel code+rwdata+rodata+init+bss 等，都计入到reserved部分。  
分配路径：

>arm64_memblock_init()

2、struct page
我们知道整个物理内存被分配为若干个页框(page frame)，一般大小位4K，一个页框对应一个`struct page`结构。`struct page`的内存占用就是在reserved部分，物理内存越大，这个区域就越大。比如2GB RAM，可能需要32MB大小的`struct page`。  
分配路径：

>__earlyonly_bootmem_alloc()

3、percpu
为所有已定义的per-cpu变量分配副本空间，静态定义的per-cpu变量越多，这个区域越大。  
分配路径：

>setup_per_cpu_areas() --> pcpu_embed_first_chunk() --> pcpu_dfl_fc_alloc()

4、devicetree
解析DTB消耗的内存  
分配路径：

>unflatten_device_tree() --> early_init_dt_alloc_memory_arch()

5、dts中reserved节点
dts中通过`reserved_memory`节点申请的reserved内存  
分配路径：

>early_init_fdt_scan_reserved_mem() --> early_init_dt_reserve_memory_arch()
