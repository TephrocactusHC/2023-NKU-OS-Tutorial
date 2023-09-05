### 关键数据结构和相关函数分析

大家在之前的实验中都尝试了`make qemu`这条指令，但实际上我们在QEMU里并没有真正模拟“硬盘”。为了实现“页面置换”的效果，我们采取的措施是，从内核的静态存储(static)区里面分出一块内存， 声称这块存储区域是”硬盘“，然后包裹一下给出”硬盘IO“的接口。
这听上去很令人费解，但如果仔细思考一下，内存和硬盘，除了一个掉电后数据易失一个不易失，一个访问快一个访问慢，其实并没有本质的区别。对于我们的页面置换算法来说，也不要求硬盘上存多余页面的交换空间能够“不易失”，反正这些页面存在内存里的时候就是易失的。理论上，我们完全可以把一块机械硬盘加以改造，写好驱动之后，插到主板的内存插槽上作为内存条使用，当然我们要忽视性能方面的差距。那么我们就把QEMU模拟出来的一块ram叫做“硬盘”，用作页面置换时的交换区，完全没有问题。你可能会觉得，这样折腾一通，我们总共能使用的页面数并没有增加，原先能直接在内存里使用的一些页面变成了“硬盘”，只是在自娱自乐。这样的想法当然是没错的，不过我们在这里只是想介绍页面置换的原理，而并不关心实际性能。

那么模拟二等这个过程我们在`driver/ide.h` `driver/ide.c` `fs/fs.h` `fs/swapfs.h` `fs/swapfs.c`通过修改代码一步步实现。

首先我们先了解一下各个文件名的含义，虽然这些文件名你可以自定义设置，但我们还是一般采用比较规范的方式进行命名,这样可以帮助你更好的理解项目结构。

`ide`在这里不是integrated development environment的意思，而是Integrated Drive Electronics的意思，表示的是一种标准的硬盘接口。我们这里写的东西和Integrated Drive Electronics并不相关，这个命名是ucore的历史遗留。

`fs`全称为file system,我们这里其实并没有“文件”的概念，这个模块称作`fs`只是说明它是“硬盘”和内核之间的接口。


```c
// kern/driver/ide.c
/*
#include"s
*/

void ide_init(void) {}

#define MAX_IDE 2
#define MAX_DISK_NSECS 56
static char ide[MAX_DISK_NSECS * SECTSIZE];

bool ide_device_valid(unsigned short ideno) { return ideno < MAX_IDE; }

size_t ide_device_size(unsigned short ideno) { return MAX_DISK_NSECS; }

int ide_read_secs(unsigned short ideno, uint32_t secno, void *dst,
                  size_t nsecs) {
    //ideno: 假设挂载了多块磁盘，选择哪一块磁盘 这里我们其实只有一块“磁盘”，这个参数就没用到
    int iobase = secno * SECTSIZE;
    memcpy(dst, &ide[iobase], nsecs * SECTSIZE);
    return 0;
}

int ide_write_secs(unsigned short ideno, uint32_t secno, const void *src,
                   size_t nsecs) { 
    int iobase = secno * SECTSIZE;
    memcpy(&ide[iobase], src, nsecs * SECTSIZE);
    return 0;
}
```

可以看到，我们这里所谓的“硬盘IO”，只是在内存里用`memcpy`把数据复制来复制去。同时为了逼真地模仿磁盘，我们只允许以磁盘扇区为数据传输的基本单位，也就是一次传输的数据必须是512字节的倍数，并且必须对齐。

```c
// kern/fs/fs.h
#ifndef __KERN_FS_FS_H__
#define __KERN_FS_FS_H__

#include <mmu.h>

#define SECTSIZE            512
#define PAGE_NSECT          (PGSIZE / SECTSIZE) //一页需要几个磁盘扇区?

#define SWAP_DEV_NO         1

#endif /* !__KERN_FS_FS_H__ */

// kern/mm/swap.h
extern size_t max_swap_offset;
// kern/mm/swap.c
size_t max_swap_offset;

// kern/fs/swapfs.c
#include <swap.h>
#include <swapfs.h>
#include <mmu.h>
#include <fs.h>
#include <ide.h>
#include <pmm.h>
#include <assert.h>

void swapfs_init(void) { //做一些检查
    static_assert((PGSIZE % SECTSIZE) == 0);
    if (!ide_device_valid(SWAP_DEV_NO)) {
        panic("swap fs isn't available.\n");
    }
    //swap.c/swap.h里的全局变量
    max_swap_offset = ide_device_size(SWAP_DEV_NO) / (PAGE_NSECT);
}

int swapfs_read(swap_entry_t entry, struct Page *page) {
    return ide_read_secs(SWAP_DEV_NO, swap_offset(entry) * PAGE_NSECT, page2kva(page), PAGE_NSECT);
}

int swapfs_write(swap_entry_t entry, struct Page *page) {
    return ide_write_secs(SWAP_DEV_NO, swap_offset(entry) * PAGE_NSECT, page2kva(page), PAGE_NSECT);
}
```
