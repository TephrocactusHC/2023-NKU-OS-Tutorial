### 项目组成

表1：实验二文件列表

```
── Makefile 
├── kern
│   ├── debug
│   │   ├── assert.h
│   │   ├── kdebug.c
│   │   ├── kdebug.h
│   │   ├── kmonitor.c
│   │   ├── kmonitor.h
│   │   ├── panic.c
│   │   └── stab.h
│   ├── driver
│   │   ├── clock.c
│   │   ├── clock.h
│   │   ├── console.c
│   │   ├── console.h
│   │   ├── intr.c
│   │   ├── intr.h
│   ├── init
│   │   ├── entry.S
│   │   └── init.c
│   ├── libs
│   │   └── stdio.c
│   ├── mm
│   │   ├── best_fit_pmm.c
│   │   ├── best_fit_pmm.h
│   │   ├── default_pmm.c
│   │   ├── default_pmm.h
│   │   ├── memlayout.h
│   │   ├── mmu.h
│   │   ├── pmm.c
│   │   └── pmm.h
│   └── trap
│       ├── trap.c
│       ├── trap.h
│       └── trapentry.S
├── libs
│   ├── atomic.h
│   ├── defs.h
│   ├── error.h
│   ├── list.h
│   ├── printfmt.c
│   ├── readline.c
│   ├── riscv.h
│   ├── sbi.c
│   ├── sbi.h
│   ├── stdarg.h
│   ├── stdio.h
│   ├── string.c
│   └── string.h
└── tools
    ├── boot.ld
    ├── function.mk
    ├── gdbinit
    ├── grade.sh
    ├── kernel.ld
    ├── kernel_nopage.ld
    ├── kflash.py
    ├── rustsbi-k210.bin
    ├── sign.c
    ├── vector.c
```



**编译方法**

编译并运行代码的命令如下：
```bash
make

make qemu
```
则可以得到如下显示界面（仅供参考）
```bash
chenyu$ make qemu
(THU.CST) os is loading ...
Special kernel symbols:
  entry  0xffffffffc0200036 (virtual)
  etext  0xffffffffc0201ad2 (virtual)
  edata  0xffffffffc0206010 (virtual)
  end    0xffffffffc0206470 (virtual)
Kernel executable memory footprint: 26KB
memory management: best_fit_pmm_manager
physcial memory map:
  memory: 0x0000000007e00000, [0x0000000080200000, 0x0000000087ffffff].
check_alloc_page() succeeded!
satp virtual address: 0xffffffffc0205000
satp physical address: 0x0000000080205000
++ setup timer interrupts
100 ticks
100 ticks
100 ticks
100 ticks
……
```
通过上图，我们可以看到ucore在显示其entry（入口地址）、etext（代码段截止处地址）、edata（数据段截止处地址）、和end（ucore截止处地址）的值后，ucore显示了物理内存的布局信息，其中包含了内存范围。接下来ucore会以页为最小分配单位实现一个简单的内存分配管理，完成页表的建立，进入分页模式，执行各种我们设置的检查，最后显示ucore建立好的页表内容，并在分页模式下响应时钟中断。
