## 附录：执行`make qemu`的大致输出
```
$ make qemu
......
(THU.CST) os is loading ...
……
check_alloc_page() succeeded!
……
check_swap() succeeded!
++ setup timer interrupts
I am No.4 philosopher_sema
Iter 1, No.4 philosopher_sema is thinking
I am No.2 philosopher_sema
Iter 1, No.2 philosopher_sema is thinking
……
Iter 1, No.0 philosopher_sema is eating
Iter 1, No.2 philosopher_sema is eating
Iter 2, No.2 philosopher_sema is thinking
……
I am No.3 philosopher_condvar
Iter 1, No.3 philosopher_condvar is thinking
……
phi_test_condvar: state_condvar[0] will eating
phi_test_condvar: signal self_cv[0] 
cond_signal begin: cvp c0443430, cvp->count 0, cvp->owner->next_count 0
cond_signal end: cvp c0443430, cvp->count 0, cvp->owner->next_count 0
Iter 1, No.0 philosopher_condvar is eating
cond_wait begin: cvp c0443458, cvp->count 0, cvp->owner->next_count 0
……
No.4 philosopher_condvar quit
No.1 philosopher_condvar quit
No.3 philosopher_condvar quit
all user-mode processes have quit.
init check memory pass.
kernel panic at kern/process/proc.c:426:
    initproc exit.
```