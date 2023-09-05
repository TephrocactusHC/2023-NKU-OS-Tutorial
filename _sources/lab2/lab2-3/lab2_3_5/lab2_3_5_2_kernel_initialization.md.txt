#### 实现分页机制
在本实验中，需要重点了解和实现基于页表的页机制和以页为单位的物理内存管理方法和分配算法等。由于ucore OS是基于80386 CPU实现的，所以CPU在进入保护模式后，就直接使能了段机制，并使得ucore OS需要在段机制的基础上建立页机制。下面比较详细地介绍了实现分页机制的过程。

接下来我们就正式开始实验啦！
首先我们要做的是内核初始化的修改，我们现在需要做的就是把原本只能直接在物理地址空间上运行的内核引入页表机制。
具体来说，我们现在想将内核代码放在虚拟地址空间中以 `0xffffffffc0200000` 开头的一段高地址空间中。那怎么做呢？首先我们需要将下面的参数修改一下：

    // tools/kernel.ld
    BASE_ADDRESS = 0xFFFFFFFFC0200000;
    //之前这里是 0x80200000

我们修改了链接脚本中的起始地址。但是这样做的话，就能从物理地址空间转移到虚拟地址空间了吗？大家可以分析一下现在我们相当于是在 bootloader 的 OpenSBI 结束后的现状，这样就可以更好的理解接下来我们需要干什么：

- 物理内存状态：OpenSBI 代码放在 `[0x80000000,0x80200000)` 中，内核代码放在以 `0x80200000` 开头的一块连续物理内存中。这个是实验一我们做完后就实现的效果。
- CPU 状态：处于 S Mode ，寄存器 `satp` 的 `MODE`被设置为 `Bare` ，即无论取指还是访存我们都通过物理地址直接访问物理内存。 `PC=0x80200000` 指向内核的第一条指令。栈顶地址 `SP` 处在 OpenSBI 代码内。
- 内核代码：这部分由于改动了链接脚本的起始地址，所以它会认为自己处在以虚拟地址 ``0xffffffffc0200000`` 开头的一段连续虚拟地址空间中，以此为依据确定代码里每个部分的地址（每一段都是从`BASE_ADDRESS`往后依次摆开的，所以代码里各段都会认为自己在`0xffffffffc0200000`之后的某个地址上，或者说编译器和链接器会把里面的符号/变量地址都对应到`0xffffffffc0200000`之后的某个地址上）

接下来，我们需要修改 ``entry.S`` 文件来实现内核的初始化，我们在入口点 ``entry.S`` 中所要做的事情是：将 `SP` 寄存器从原先指向OpenSBI 某处的栈空间，改为指向我们自己在内核的内存空间里分配的栈；同时需要跳转到函数 `kern_init` 中。

在之前的实验中，我们已经在 `entry.S` 自己分配了一块 `16KiB`的内存用来做启动栈：

```asm
#include <mmu.h>
#include <memlayout.h>

    .section .text,"ax",%progbits
    .globl kern_entry
kern_entry:
    la sp, bootstacktop

    tail kern_init

.section .data
    # .align 2^12
    .align PGSHIFT
    .global bootstack
bootstack:
    .space KSTACKSIZE
    .global bootstacktop
bootstacktop:
```

通过之前的实验大家应该都明白：符号 `bootstacktop` 就是我们需要的栈顶地址, 符号 `kern_init` 代表了我们要跳转到的地址。之前我们直接将 `bootstacktop` 的值给到 `SP` ， 再跳转到 `rust_main` 就行了。看上去上面的这个代码也能够实现我们想要的初始化效果呀？但问题在于，由于我们修改了链接脚本的起始地址，编译器和链接器认为内核开头地址为 ``0xffffffffc0200000``，因此这两个符号会被翻译成比这个开头地址还要高的某个虚拟地址。而我们的 CPU 目前还处于 `Bare` 模式，会将地址都当成物理地址处理。这样，我们跳转到 `rust_main` 就会跳转到比`0xffffffffc0200000`还大的一个物理地址。但物理地址显然不可能有这么多位！这就会出现问题。

于是，我们需要想办法利用刚学的页表知识，帮内核将需要的虚拟地址空间构造出来。也就是：构建一个合适的页表，让`satp`指向这个页表，然后使用地址的时候都要经过这个页表的翻译，使得虚拟地址`0xFFFFFFFFC0200000`经过页表的翻译恰好变成`0x80200000`，这个地址显然就比较合适了，也就不会出错了。

理论知识告诉我们，所有的虚拟地址有一个固定的偏移量。而要想实现页表结构这个偏移量显然是不可或缺的。而虚拟地址和物理地址之间的差值就可以当成是这个偏移量。

比如内核的第一条指令，虚拟地址为 `0xffffffffc0200000` ，物理地址为 `0x80200000` ，因此，我们只要将虚拟地址减去 `0xffffffff40000000` ，就得到了物理地址。所以当我们需要做到去访问内核里面的一个物理地址 `va` 时，而已知虚拟地址为 `va` 时，则 `va` 处的代码或数据就放在物理地址为 `pa = va - 0xffffffff40000000` 处的物理内存中，我们真正所要做的是要让 CPU 去访问 `pa`。因此，我们要通过恰当构造页表，来对于内核所属的虚拟地址，实现这种 `va` 到 `pa` 的映射。

还记得之前的理论介绍的内容吗？那时我们提到，将一个三级页表项的标志位 `R,W,X` 不设为全 `0` ，可以将它变为一个叶子，从而获得大小为 `1GiB` 的一个大页。

我们假定内核大小不超过 `1GiB`，通过一个大页将虚拟地址区间`[0xffffffffc0000000,0xffffffffffffffff]` 映射到物理地址区间 `[0x80000000,0xc0000000)`，而我们只需要分配一页内存用来存放三级页表，并将其最后一个页表项(也就是对应我们使用的虚拟地址区间的页表项)进行适当设置即可。对应的代码如下所示：

```asm
#include <mmu.h>
#include <memlayout.h>

    .section .text,"ax",%progbits
    .globl kern_entry
kern_entry:
    # t0 := 三级页表的虚拟地址
    lui     t0, %hi(boot_page_table_sv39)
    # t1 := 0xffffffff40000000 即虚实映射偏移量
    li      t1, 0xffffffffc0000000 - 0x80000000
    # t0 减去虚实映射偏移量 0xffffffff40000000，变为三级页表的物理地址
    sub     t0, t0, t1
    # t0 >>= 12，变为三级页表的物理页号
    srli    t0, t0, 12

    # t1 := 8 << 60，设置 satp 的 MODE 字段为 Sv39
    li      t1, 8 << 60
    # 将刚才计算出的预设三级页表物理页号附加到 satp 中
    or      t0, t0, t1
    # 将算出的 t0(即新的MODE|页表基址物理页号) 覆盖到 satp 中
    csrw    satp, t0
    # 使用 sfence.vma 指令刷新 TLB
    sfence.vma
    # 从此，我们给内核搭建出了一个完美的虚拟内存空间！
    #nop # 可能映射的位置有些bug。。插入一个nop
    
    # 我们在虚拟内存空间中：随意将 sp 设置为虚拟地址！
    lui sp, %hi(bootstacktop)

    # 我们在虚拟内存空间中：随意跳转到虚拟地址！
    # 跳转到 kern_init
    lui t0, %hi(kern_init)
    addi t0, t0, %lo(kern_init)
    jr t0

.section .data
    # .align 2^12
    .align PGSHIFT
    .global bootstack
bootstack:
    .space KSTACKSIZE
    .global bootstacktop
bootstacktop:

.section .data
    # 由于我们要把这个页表放到一个页里面，因此必须 12 位对齐
    .align PGSHIFT
    .global boot_page_table_sv39
# 分配 4KiB 内存给预设的三级页表
boot_page_table_sv39:
    # 0xffffffff_c0000000 map to 0x80000000 (1G)
    # 前 511 个页表项均设置为 0 ，因此 V=0 ，意味着是空的(unmapped)
    .zero 8 * 511
    # 设置最后一个页表项，PPN=0x80000，标志位 VRWXAD 均为 1
    .quad (0x80000 << 10) | 0xcf # VRWXAD
```

总结一下，要进入虚拟内存访问方式，需要如下步骤：

1. 分配页表所在内存空间并初始化页表；
2. 设置好页基址寄存器（指向页表起始地址）；
3. 刷新 TLB。

到现在为止，看上去复杂无比的虚拟内存空间，我们终于得以窥视一二了。
