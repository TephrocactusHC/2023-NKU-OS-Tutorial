### 滴答滴答（时钟中断）

时钟中断需要CPU硬件的支持。CPU以"时钟周期"为工作的基本时间单位，对逻辑门的时序电路进行同步。

我们的“时钟中断”实际上就是”每隔若干个时钟周期执行一次的程序“。

”若干个时钟周期“是多少个？太短了肯定不行。如果时钟中断处理程序需要100个时钟周期执行，而你每50个时钟周期就触发一个时钟中断，那么间隔时间连一个完整的时钟中断程序都跑不完。如果你200个时钟周期就触发一个时钟中断，那么CPU的时间将有一半消耗在时钟中断，开销太大。一般而言，可以设置时钟中断间隔设置为CPU频率的1%，也就是每秒钟触发100次时钟中断，避免开销过大。

我们用到的RISCV对时钟中断的硬件支持包括：

- OpenSBI提供的`sbi_set_timer()`接口，可以传入一个时刻，让它在那个时刻触发一次时钟中断
- `rdtime`伪指令，读取一个叫做`time`的CSR的数值，表示CPU启动之后经过的真实时间。在不同硬件平台，时钟频率可能不同。在QEMU上，这个时钟的频率是10MHz, 每过1s, `rdtime`返回的结果增大`10000000`

> 趣闻 
>
> 在RISCV32和RISCV64架构中，`time`寄存器都是64位的。
>
> `rdcycle`伪指令可以读取经过的时钟周期数目，对应一个寄存器`cycle`

注意，我们需要“每隔若干时间就发生一次时钟中断”，但是OpenSBI提供的接口一次只能设置一个时钟中断事件。我们采用的方式是：一开始只设置一个时钟中断，之后每次发生时钟中断的时候，设置下一次的时钟中断。

在clock.c里面初始化时钟并封装一些接口

```c
//libs/sbi.c

//当time寄存器(rdtime的返回值)为stime_value的时候触发一个时钟中断
void sbi_set_timer(unsigned long long stime_value) {
    sbi_call(SBI_SET_TIMER, stime_value, 0, 0);
}

// kern/driver/clock.c
#include <clock.h>
#include <defs.h>
#include <sbi.h>
#include <stdio.h>
#include <riscv.h>

//volatile告诉编译器这个变量可能在其他地方被瞎改一通，所以编译器不要对这个变量瞎优化
volatile size_t ticks;

//对64位和32位架构，读取time的方法是不同的
//32位架构下，需要把64位的time寄存器读到两个32位整数里，然后拼起来形成一个64位整数
//64位架构简单的一句rdtime就可以了
//__riscv_xlen是gcc定义的一个宏，可以用来区分是32位还是64位。
static inline uint64_t get_time(void) {//返回当前时间
#if __riscv_xlen == 64
    uint64_t n;
    __asm__ __volatile__("rdtime %0" : "=r"(n));
    return n;
#else
    uint32_t lo, hi, tmp;
    __asm__ __volatile__(
        "1:\n"
        "rdtimeh %0\n"
        "rdtime %1\n"
        "rdtimeh %2\n"
        "bne %0, %2, 1b"
        : "=&r"(hi), "=&r"(lo), "=&r"(tmp));
    return ((uint64_t)hi << 32) | lo;
#endif
}


// Hardcode timebase
static uint64_t timebase = 100000;

void clock_init(void) {
    // sie这个CSR可以单独使能/禁用某个来源的中断。默认时钟中断是关闭的
    // 所以我们要在初始化的时候，使能时钟中断
    set_csr(sie, MIP_STIP); // enable timer interrupt in sie
    //设置第一个时钟中断事件
    clock_set_next_event();
    // 初始化一个计数器
    ticks = 0;

    cprintf("++ setup timer interrupts\n");
}
//设置时钟中断：timer的数值变为当前时间 + timebase 后，触发一次时钟中断
//对于QEMU, timer增加1，过去了10^-7 s， 也就是100ns
void clock_set_next_event(void) { sbi_set_timer(get_time() + timebase); }
```

回来看trap.c里面时钟中断处理的代码, 还是很简单的：每秒100次时钟中断，触发每次时钟中断后设置10ms后触发下一次时钟中断，每触发100次时钟中断（1秒钟）输出一行信息到控制台。

```c
// kern/trap/trap.c
#include<clock.h>

#define TICK_NUM 100
static void print_ticks() {
    cprintf("%d ticks\n", TICK_NUM);
#ifdef DEBUG_GRADE
    cprintf("End of Test.\n");
    panic("EOT: kernel seems ok.");
#endif
}

void interrupt_handler(struct trapframe *tf) {
    intptr_t cause = (tf->cause << 1) >> 1;
    switch (cause) {
       	/* blabla 其他case*/
        case IRQ_S_TIMER:
            clock_set_next_event();//发生这次时钟中断的时候，我们要设置下一次时钟中断
            if (++ticks % TICK_NUM == 0) {
                print_ticks();
            }
            break;
        /* blabla 其他case*/
}
```

现在执行`make qemu`, 应该能看到打印一行行的`100 ticks`

目前为止的代码可以在[这里](https://github.com/Liurunda/riscv64-ucore/tree/lab1/lab1)找到，遇到困难可以参考。