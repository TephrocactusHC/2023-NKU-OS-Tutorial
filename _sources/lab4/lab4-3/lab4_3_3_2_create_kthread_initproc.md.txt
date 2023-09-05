#### 创建第 1 个内核线程 initproc 

第0个内核线程主要工作是完成内核中各个子系统的初始化，然后就通过执行cpu\_idle函数开始过退休生活了。所以uCore接下来还需创建其他进程来完成各种工作，但idleproc内核子线程自己不想做，于是就通过调用kernel\_thread函数创建了一个内核线程init\_main。在实验四中，这个子内核线程的工作就是输出一些字符串，然后就返回了（参看init\_main函数）。但在后续的实验中，init\_main的工作就是创建特定的其他内核线程或用户进程（实验五涉及）。下面我们来分析一下创建内核线程的函数kernel\_thread：

```
int kernel_thread(int (*fn)(void *), void *arg, uint32_t clone_flags) {
    // 对trameframe，也就是我们程序的一些上下文进行一些初始化
    struct trapframe tf;
    memset(&tf, 0, sizeof(struct trapframe));

    // 设置内核线程的参数和函数指针
    tf.gpr.s0 = (uintptr_t)fn; // s0 寄存器保存函数指针
    tf.gpr.s1 = (uintptr_t)arg; // s1 寄存器保存函数参数

    // 设置 trapframe 中的 status 寄存器（SSTATUS）
    // SSTATUS_SPP：Supervisor Previous Privilege（设置为 supervisor 模式，因为这是一个内核线程）
    // SSTATUS_SPIE：Supervisor Previous Interrupt Enable（设置为启用中断，因为这是一个内核线程）
    // SSTATUS_SIE：Supervisor Interrupt Enable（设置为禁用中断，因为我们不希望该线程被中断）
    tf.status = (read_csr(sstatus) | SSTATUS_SPP | SSTATUS_SPIE) & ~SSTATUS_SIE;

    // 将入口点（epc）设置为 kernel_thread_entry 函数，作用实际上是将pc指针指向它(*trapentry.S会用到)
    tf.epc = (uintptr_t)kernel_thread_entry;

    // 使用 do_fork 创建一个新进程（内核线程），这样才真正用设置的tf创建新进程。
    return do_fork(clone_flags | CLONE_VM, 0, &tf);
}
```

注意，kernel\_thread函数采用了局部变量tf来放置保存内核线程的临时中断帧，并把中断帧的指针传递给do\_fork函数，而do\_fork函数会调用copy\_thread函数来在新创建的进程内核栈上专门给进程的中断帧分配一块空间。

给中断帧分配完空间后，就需要构造新进程的中断帧，具体过程是：首先给tf进行清零初始化，随后设置设置内核线程的参数和函数指针。要特别注意对tf.status的赋值过程，其读取sstatus寄存器的值，然后根据特定的位操作，设置SPP和SPIE位，并同时清除SIE位，从而实现特权级别切换、保留中断使能状态并禁用中断的操作。而initproc内核线程从哪里开始执行呢？tf.epc的指出了是kernel\_thread\_entry（位于kern/process/entry.S中），kernel\_thread\_entry是entry.S中实现的汇编函数，它做的事情很简单：

```
kernel_thread_entry: # void kernel_thread(void)
pushl %edx # push arg
call *%ebx # call fn
pushl %eax # save the return value of fn(arg)
call do_exit # call do_exit to terminate current thread
```

从上可以看出，kernel\_thread\_entry函数主要为内核线程的主体fn函数做了一个准备开始和结束运行的“壳”，并把函数fn的参数arg（保存在edx寄存器中）压栈，然后调用fn函数，把函数返回值eax寄存器内容压栈，调用do\_exit函数退出线程执行。

do\_fork是创建线程的主要函数。kernel\_thread函数通过调用do\_fork函数最终完成了内核线程的创建工作。下面我们来分析一下do\_fork函数的实现（练习2）。do\_fork函数主要做了以下7件事情：

1. 分配并初始化进程控制块（`alloc_proc`函数）
2. 分配并初始化内核栈（`setup_stack`函数）
3. 根据`clone_flags`决定是复制还是共享内存管理系统（`copy_mm`函数）
4. 设置进程的中断帧和上下文（`copy_thread`函数）
5. 把设置好的进程加入链表
6. 将新建的进程设为就绪态
7. 将返回值设为线程id

这里需要注意的是，如果上述前3步执行没有成功，则需要做对应的出错处理，把相关已经占有的内存释放掉。copy\_mm函数目前只是把current-\>mm设置为NULL，这是由于目前在实验四中只能创建内核线程，proc-\>mm描述的是进程用户态空间的情况，所以目前mm还用不上。

在这里我们需要尤其关注`copy_thread`函数：

```c
static void
copy_thread(struct proc_struct *proc, uintptr_t esp, struct trapframe *tf) {
    proc->tf = (struct trapframe *)(proc->kstack + KSTACKSIZE - sizeof(struct trapframe));
    *(proc->tf) = *tf;

    // Set a0 to 0 so a child process knows it's just forked
    proc->tf->gpr.a0 = 0;
    proc->tf->gpr.sp = (esp == 0) ? (uintptr_t)proc->tf : esp;

    proc->context.ra = (uintptr_t)forkret;
    proc->context.sp = (uintptr_t)(proc->tf);
}
```

在这里我们首先在上面分配的内核栈上分配出一片空间来保存`trapframe`。然后，我们将`trapframe`中的`a0`寄存器（返回值）设置为0，说明这个进程是一个子进程。之后我们将上下文中的`ra`设置为了`forkret`函数的入口，并且把`trapframe`放在上下文的栈顶。在下一个小节，我们会看到这么做之后ucore是如何完成进程切换的。