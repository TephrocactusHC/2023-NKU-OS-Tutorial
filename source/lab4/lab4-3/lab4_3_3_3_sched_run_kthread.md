#### 调度并执行内核线程 initproc 

在uCore执行完proc\_init函数后，就创建好了两个内核线程：idleproc和initproc，这时uCore当前的执行现场就是idleproc，等到执行到init函数的最后一个函数cpu\_idle之前，uCore的所有初始化工作就结束了，idleproc将通过执行cpu\_idle函数让出CPU，给其它内核线程执行，具体过程如下：

```
void
cpu_idle(void) {
	while (1) {
		if (current->need_resched) {
			schedule();
			……
```

首先，判断当前内核线程idleproc的need\_resched是否不为0，回顾前面“创建第一个内核线程idleproc”中的描述，proc\_init函数在初始化idleproc中，就把idleproc-\>need\_resched置为1了，所以会马上调用schedule函数找其他处于“就绪”态的进程执行。

uCore在实验四中只实现了一个最简单的FIFO调度器，其核心就是schedule函数。它的执行逻辑很简单：

1．设置当前内核线程current-\>need\_resched为0；
2．在proc\_list队列中查找下一个处于“就绪”态的线程或进程next；
3．找到这样的进程后，就调用proc\_run函数，保存当前进程current的执行现场（进程上下文），恢复新进程的执行现场，完成进程切换。

至此，新的进程next就开始执行了。由于在proc10中只有两个内核线程，且idleproc要让出CPU给initproc执行，我们可以看到schedule函数通过查找proc\_list进程队列，只能找到一个处于“就绪”态的initproc内核线程。并通过proc\_run和进一步的switch\_to函数完成两个执行现场的切换，具体流程如下：

1. 将当前运行的进程设置为要切换过去的进程
2. 将页表换成新进程的页表
3. 使用`switch_to`切换到新进程

第三步proc\_run函数调用switch\_to函数，参数是前一个进程和后一个进程的执行现场：process context。在上一节“设计进程控制块”中，描述了context结构包含的要保存和恢复的寄存器。我们再看看switch.S中的switch\_to函数的执行流程：

```assembly
.text
# void switch_to(struct proc_struct* from, struct proc_struct* to)
.globl switch_to
switch_to:
    # save from's registers
    STORE ra, 0*REGBYTES(a0)
    STORE sp, 1*REGBYTES(a0)
    STORE s0, 2*REGBYTES(a0)
    STORE s1, 3*REGBYTES(a0)
    STORE s2, 4*REGBYTES(a0)
    STORE s3, 5*REGBYTES(a0)
    STORE s4, 6*REGBYTES(a0)
    STORE s5, 7*REGBYTES(a0)
    STORE s6, 8*REGBYTES(a0)
    STORE s7, 9*REGBYTES(a0)
    STORE s8, 10*REGBYTES(a0)
    STORE s9, 11*REGBYTES(a0)
    STORE s10, 12*REGBYTES(a0)
    STORE s11, 13*REGBYTES(a0)

    # restore to's registers
    LOAD ra, 0*REGBYTES(a1)
    LOAD sp, 1*REGBYTES(a1)
    LOAD s0, 2*REGBYTES(a1)
    LOAD s1, 3*REGBYTES(a1)
    LOAD s2, 4*REGBYTES(a1)
    LOAD s3, 5*REGBYTES(a1)
    LOAD s4, 6*REGBYTES(a1)
    LOAD s5, 7*REGBYTES(a1)
    LOAD s6, 8*REGBYTES(a1)
    LOAD s7, 9*REGBYTES(a1)
    LOAD s8, 10*REGBYTES(a1)
    LOAD s9, 11*REGBYTES(a1)
    LOAD s10, 12*REGBYTES(a1)
    LOAD s11, 13*REGBYTES(a1)

    ret
```

可以看出来这段代码就是将需要保存的寄存器进行保存和调换。其中的a0和a1是RISC-V 架构中通用寄存器，它们用于传递参数，也就是说a0指向原进程，a1指向目的进程。

在之前我们也已经谈到过了，这里只需要调换被调用者保存寄存器即可。由于我们在初始化时把上下文的`ra`寄存器设定成了`forkret`函数的入口，所以这里会返回到`forkret`函数。`forkrets`函数很短，位于kern/trap/trapentry.S：

```assembly
    .globl forkrets
forkrets:
    # set stack to this new process's trapframe
    move sp, a0
    j __trapret
```

这里把传进来的参数，也就是进程的中断帧放在了`sp`，这样在`__trapret`中就可以直接从中断帧里面恢复所有的寄存器啦！我们在初始化的时候对于中断帧做了一点手脚，`epc`寄存器指向的是`kernel_thread_entry`，`s0`寄存器里放的是新进程要执行的函数，`s1`寄存器里放的是传给函数的参数。在`kernel_thread_entry`函数中：

```assembly
.text
.globl kernel_thread_entry
kernel_thread_entry:        # void kernel_thread(void)
	move a0, s1
	jalr s0

	jal do_exit
```

我们把参数放在了`a0`寄存器，并跳转到`s0`执行我们指定的函数！这样，一个进程的初始化就完成了。至此，我们实现了基本的进程管理，并且成功创建并切换到了我们的第一个内核进程。

