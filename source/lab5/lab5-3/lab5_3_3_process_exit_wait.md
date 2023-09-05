### 进程退出和等待进程 

当进程执行完它的工作后，就需要执行退出操作，释放进程占用的资源。ucore分了两步来完成这个工作，首先由进程本身完成大部分资源的占用内存回收工作，然后由此进程的父进程完成剩余资源占用内存的回收工作。为何不让进程本身完成所有的资源回收工作呢？这是因为进程要执行回收操作，就表明此进程还存在，还在执行指令，这就需要内核栈的空间不能释放，且表示进程存在的进程控制块不能释放。所以需要父进程来帮忙释放子进程无法完成的这两个资源回收工作。

为此在用户态的函数库中提供了exit函数，此函数最终访问sys\_exit系统调用接口让操作系统来帮助当前进程执行退出过程中的部分资源回收。我们来看看ucore是如何做进程退出工作的。

```c
// /user/libs/ulib.c

void
exit(int error_code) {
    sys_exit(error_code);
    cprintf("BUG: exit failed.\n");
    while (1);
}
```

首先，exit函数会把一个退出码error\_code传递给ucore，ucore通过执行位于`/proc/process/proc.c`中的内核函数`do_exit`来完成对当前进程的退出处理，主要工作简单地说就是回收当前进程所占的大部分内存资源，并通知父进程完成最后的回收工作，具体流程如下：

```c
// /proc/process/proc.c

int
do_exit(int error_code) {
    // 检查当前进程是否为idleproc或initproc，如果是，发出panic
    if (current == idleproc) {
        panic("idleproc exit.\n");
    }
    if (current == initproc) {
        panic("initproc exit.\n");
    }

    // 获取当前进程的内存管理结构mm
    struct mm_struct *mm = current->mm;

    // 如果mm不为空，说明是用户进程
    if (mm != NULL) {
        // 切换到内核页表，确保接下来的操作在内核空间执行
        lcr3(boot_cr3);

        // 如果mm引用计数减到0，说明没有其他进程共享此mm
        if (mm_count_dec(mm) == 0) {
            // 释放用户虚拟内存空间相关的资源
            exit_mmap(mm);
            put_pgdir(mm);
            mm_destroy(mm);
        }
        // 将当前进程的mm设置为NULL，表示资源已经释放
        current->mm = NULL;
    }

    // 设置进程状态为PROC_ZOMBIE，表示进程已退出
    current->state = PROC_ZOMBIE;
    current->exit_code = error_code;

    bool intr_flag;
    struct proc_struct *proc;

    // 关中断
    local_intr_save(intr_flag);
    {
        // 获取当前进程的父进程
        proc = current->parent;
        
        // 如果父进程处于等待子进程状态，则唤醒父进程
        if (proc->wait_state == WT_CHILD) {
            wakeup_proc(proc);
        }
        
        // 遍历当前进程的所有子进程
        while (current->cptr != NULL) {
            proc = current->cptr;
            current->cptr = proc->optr;

            // 设置子进程的父进程为initproc，并加入initproc的子进程链表
            proc->yptr = NULL;
            if ((proc->optr = initproc->cptr) != NULL) {
                initproc->cptr->yptr = proc;
            }
            proc->parent = initproc;
            initproc->cptr = proc;

            // 如果子进程也处于退出状态，唤醒initproc
            if (proc->state == PROC_ZOMBIE) {
                if (initproc->wait_state == WT_CHILD) {
                    wakeup_proc(initproc);
                }
            }
        }
    }
    // 开中断
    local_intr_restore(intr_flag);

    // 调用调度器，选择新的进程执行
    schedule();

    // 如果执行到这里，表示代码执行出现错误，发出panic
    panic("do_exit will not return!! %d.\n", current->pid);
}

```

下面我们来看这些系统调用的实现。