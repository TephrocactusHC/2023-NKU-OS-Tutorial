#### 物理内存探测的设计思路

操作系统怎样知道物理内存所在的那段物理地址呢？在 RISC-V 中，这个一般是由 bootloader ，即 OpenSBI 来完成的。它来完成对于包括物理内存在内的各外设的扫描，将扫描结果以 DTB(Device Tree Blob) 的格式保存在物理内存中的某个地方。随后 OpenSBI 会将其地址保存在 `a1` 寄存器中，给我们使用。

这个扫描结果描述了所有外设的信息，当中也包括 Qemu 模拟的 RISC-V 计算机中的物理内存。

> 扩展 **Qemu 模拟的 RISC-V virt 计算机中的物理内存**
>
> 通过查看[virt.c](https://github.com/qemu/qemu/blob/master/hw/riscv/virt.c)的**virt_memmap[]**的定义，可以了解到 Qemu 模拟的 RISC-V virt 计算机的详细物理内存布局。可以看到，整个物理内存中有不少内存空洞（即含义为**unmapped**的地址空间），也有很多外设特定的地址空间，现在我们看不懂没有关系，后面会慢慢涉及到。目前只需关心最后一块含义为**DRAM**的地址空间，这就是 OS 将要管理的 128MB 的内存空间。
>
> | 起始地址   | 终止地址   | 含义                                                  |
> | :--------- | :--------- | :---------------------------------------------------- |
> | 0x0        | 0x100      | QEMU VIRT_DEBUG                                       |
> | 0x100      | 0x1000     | unmapped                                              |
> | 0x1000     | 0x12000    | QEMU MROM (包括 hard-coded reset vector; device tree) |
> | 0x12000    | 0x100000   | unmapped                                              |
> | 0x100000   | 0x101000   | QEMU VIRT_TEST                                        |
> | 0x101000   | 0x2000000  | unmapped                                              |
> | 0x2000000  | 0x2010000  | QEMU VIRT_CLINT                                       |
> | 0x2010000  | 0x3000000  | unmapped                                              |
> | 0x3000000  | 0x3010000  | QEMU VIRT_PCIE_PIO                                    |
> | 0x3010000  | 0xc000000  | unmapped                                              |
> | 0xc000000  | 0x10000000 | QEMU VIRT_PLIC                                        |
> | 0x10000000 | 0x10000100 | QEMU VIRT_UART0                                       |
> | 0x10000100 | 0x10001000 | unmapped                                              |
> | 0x10001000 | 0x10002000 | QEMU VIRT_VIRTIO                                      |
> | 0x10002000 | 0x20000000 | unmapped                                              |
> | 0x20000000 | 0x24000000 | QEMU VIRT_FLASH                                       |
> | 0x24000000 | 0x30000000 | unmapped                                              |
> | 0x30000000 | 0x40000000 | QEMU VIRT_PCIE_ECAM                                   |
> | 0x40000000 | 0x80000000 | QEMU VIRT_PCIE_MMIO                                   |
> | 0x80000000 | 0x88000000 | DRAM 缺省 128MB，大小可配置                           |

不过为了简单起见，我们并不打算自己去解析这个结果。因为我们知道，Qemu 规定的 DRAM 物理内存的起始物理地址为 `0x80000000` 。而在 Qemu 中，可以使用 `-m` 指定 RAM 的大小，默认是 `128MiB` 。因此，默认的 DRAM 物理内存地址范围就是 `[0x80000000,0x88000000)` 。我们直接将 DRAM 物理内存结束地址硬编码到内核中：

```c
// kern/mm/memlayout.h

#define KERNBASE            0xFFFFFFFFC0200000 
#define KMEMSIZE            0x7E00000          
#define KERNTOP             (KERNBASE + KMEMSIZE) 

#define PHYSICAL_MEMORY_END         0x88000000
#define PHYSICAL_MEMORY_OFFSET      0xFFFFFFFF40000000 //物理地址和虚拟地址的偏移量
#define KERNEL_BEGIN_PADDR          0x80200000
#define KERNEL_BEGIN_VADDR          0xFFFFFFFFC0200000
```

但是，有一部分 DRAM 空间已经被占用，不能用来存别的东西了！

- 物理地址空间 `[0x80000000,0x80200000)` 被 OpenSBI 占用；
- 物理地址空间 `[0x80200000,KernelEnd)` 被内核各代码与数据段占用；
- 其实设备树扫描结果 DTB 还占用了一部分物理内存，不过由于我们不打算使用它，所以可以将它所占用的空间用来存别的东西。

于是，我们可以用来存别的东西的物理内存的物理地址范围是：`[KernelEnd, 0x88000000)` 。这里的 `KernelEnd` 为内核代码结尾的物理地址。在 `kernel.ld` 中定义的 `end` 符号为内核代码结尾的虚拟地址。

为了管理物理内存，我们需要在内核里定义一些数据结构，来存储”当前使用了哪些物理页面，哪些物理页面没被使用“这样的信息，使用的是Page结构体。我们将一些Page结构体在内存里排列在内核后面，这要占用一些内存。而摆放这些Page结构体的物理页面，以及内核占用的物理页面，之后都无法再使用了。我们用`page_init()`函数给这些管理物理内存的结构体做初始化。下面是代码：

```c
// kern/mm/pmm.h

/* *
 * PADDR - takes a kernel virtual address (an address that points above
 * KERNBASE),
 * where the machine's maximum 256MB of physical memory is mapped and returns
 * the
 * corresponding physical address.  It panics if you pass it a non-kernel
 * virtual address.
 * */
#define PADDR(kva)                                                 \
    ({                                                             \
        uintptr_t __m_kva = (uintptr_t)(kva);                      \
        if (__m_kva < KERNBASE) {                                  \
            panic("PADDR called with invalid kva %08lx", __m_kva); \
        }                                                          \
        __m_kva - va_pa_offset;                                    \
    })

/* *
 * KADDR - takes a physical address and returns the corresponding kernel virtual
 * address. It panics if you pass an invalid physical address.
 * */
/*
#define KADDR(pa)                                                \
    ({                                                           \
        uintptr_t __m_pa = (pa);                                 \
        size_t __m_ppn = PPN(__m_pa);                            \
        if (__m_ppn >= npage) {                                  \
            panic("KADDR called with invalid pa %08lx", __m_pa); \
        }                                                        \
        (void *)(__m_pa + va_pa_offset);                         \
    })
*/
extern struct Page *pages;
extern size_t npage;

```
```c
// kern/mm/pmm.c

// pages指针保存的是第一个Page结构体所在的位置，也可以认为是Page结构体组成的数组的开头
// 由于C语言的特性，可以把pages作为数组名使用，pages[i]表示顺序排列的第i个结构体
struct Page *pages;
size_t npage = 0;
uint64_t va_pa_offset;
// memory starts at 0x80000000 in RISC-V
const size_t nbase = DRAM_BASE / PGSIZE;
//(npage - nbase)表示物理内存的页数

static void page_init(void) {
    va_pa_offset = PHYSICAL_MEMORY_OFFSET; //硬编码 0xFFFFFFFF40000000

    uint64_t mem_begin = KERNEL_BEGIN_PADDR;//硬编码 0x80200000
    uint64_t mem_size = PHYSICAL_MEMORY_END - KERNEL_BEGIN_PADDR;
    uint64_t mem_end = PHYSICAL_MEMORY_END; //硬编码 0x88000000

    cprintf("physcial memory map:\n");
    cprintf("  memory: 0x%016lx, [0x%016lx, 0x%016lx].\n", mem_size, mem_begin,
            mem_end - 1);

    uint64_t maxpa = mem_end;

    if (maxpa > KERNTOP) {
        maxpa = KERNTOP;
    }

    npage = maxpa / PGSIZE;
    
    extern char end[];
    pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);
    //把pages指针指向内核所占内存空间结束后的第一页

    //一开始把所有页面都设置为保留给内核使用的，之后再设置哪些页面可以分配给其他程序
    for (size_t i = 0; i < npage - nbase; i++) {
        SetPageReserved(pages + i);//记得吗？在kern/mm/memlayout.h定义的
    }
	//从这个地方开始才是我们可以自由使用的物理内存
    uintptr_t freemem = PADDR((uintptr_t)pages + sizeof(struct Page) * (npage - nbase));
	//按照页面大小PGSIZE进行对齐, ROUNDUP, ROUNDDOWN是在libs/defs.h定义的
    mem_begin = ROUNDUP(freemem, PGSIZE);
    mem_end = ROUNDDOWN(mem_end, PGSIZE);
    if (freemem < mem_end) {
        //初始化我们可以自由使用的物理内存
        init_memmap(pa2page(mem_begin), (mem_end - mem_begin) / PGSIZE);
    }
}
```

在`page_init()`的代码里，我们调用了一个函数`init_memmap()`, 这和我们的另一个结构体`pmm_manager`有关。虽然C语言基本上不支持面向对象，但我们可以用类似面向对象的思路，把”物理内存管理“的功能集中给一个结构体。我们甚至可以让函数指针作为结构体的成员，强行在C语言里支持了”成员函数“。可以看到，我们调用的`init_memmap()`实际上又调用了`pmm_manager`的一个”成员函数“。

```c
// kern/mm/pmm.c

// physical memory management
const struct pmm_manager *pmm_manager;


// init_memmap - call pmm->init_memmap to build Page struct for free memory
static void init_memmap(struct Page *base, size_t n) {
    pmm_manager->init_memmap(base, n);
}
```

```c
// kern/mm/pmm.h
#ifndef __KERN_MM_PMM_H__
#define __KERN_MM_PMM_H__

#include <assert.h>
#include <atomic.h>
#include <defs.h>
#include <memlayout.h>
#include <mmu.h>
#include <riscv.h>

// pmm_manager is a physical memory management class. A special pmm manager -
// XXX_pmm_manager
// only needs to implement the methods in pmm_manager class, then
// XXX_pmm_manager can be used
// by ucore to manage the total physical memory space.
struct pmm_manager {
    const char *name;  // XXX_pmm_manager's name
    void (*init)(
        void);  // 初始化XXX_pmm_manager内部的数据结构（如空闲页面的链表）
    void (*init_memmap)(
        struct Page *base,
        size_t n);  //知道了可用的物理页面数目之后，进行更详细的初始化
    struct Page *(*alloc_pages)(
        size_t n);  // 分配至少n个物理页面, 根据分配算法可能返回不同的结果
    void (*free_pages)(struct Page *base, size_t n);  // free >=n pages with
                                                      // "base" addr of Page
                                                      // descriptor
                                                      // structures(memlayout.h)
    size_t (*nr_free_pages)(void);  // 返回空闲物理页面的数目
    void (*check)(void);            // 测试正确性
};

extern const struct pmm_manager *pmm_manager;

void pmm_init(void);

struct Page *alloc_pages(size_t n);
void free_pages(struct Page *base, size_t n);
size_t nr_free_pages(void); // number of free pages

#define alloc_page() alloc_pages(1)
#define free_page(page) free_pages(page, 1)
```

pmm_manager提供了各种接口：分配页面，释放页面，查看当前空闲页面数。但是我们好像始终没看见pmm_manager内部对这些接口的实现，其实是因为那些接口只是作为函数指针，作为pmm_manager的一部分，我们需要把那些函数指针变量赋值为真正的函数名称。

还记得最早我们在`pmm_init()`里首先调用了`init_pmm_manager()`, 在这里面我们把pmm_manager的指针赋值成`&default_pmm_manager`， 看起来我们在这里实现了那些接口。

```c
// init_pmm_manager - initialize a pmm_manager instance
static void init_pmm_manager(void) {
    pmm_manager = &default_pmm_manager;
    cprintf("memory management: %s\n", pmm_manager->name);
    pmm_manager->init();
}
// alloc_pages - call pmm->alloc_pages to allocate a continuous n*PAGESIZE
// memory
struct Page *alloc_pages(size_t n) {
    // 在这里编写你的物理内存分配算法。
    // 你可以参考nr_free_pages() 函数进行设计，
    // 了解物理内存管理器的工作原理，然后在这里实现自己的分配算法。
    // 实现算法后，调用 pmm_manager->alloc_pages(n) 来分配物理内存，
    // 然后返回分配的 Page 结构指针。
}

// free_pages - call pmm->free_pages to free a continuous n*PAGESIZE memory
void free_pages(struct Page *base, size_t n) {
    // 在这里编写你的物理内存释放算法。
    // 你可以参考nr_free_pages() 函数进行设计，
    // 了解物理内存管理器的工作原理，然后在这里实现自己的释放算法。
    // 实现算法后，调用 pmm_manager->free_pages(base, n) 来释放物理内存。
}

// nr_free_pages - call pmm->nr_free_pages to get the size (nr*PAGESIZE)
// of current free memory
size_t nr_free_pages(void) {
    size_t ret;
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        ret = pmm_manager->nr_free_pages();
    }
    local_intr_restore(intr_flag);
    return ret;
}
```

到现在，我们距离完整的内存管理， 就只差`default_pmm_manager`结构体的实现了，也就是我们要在里面实现页面分配算法。