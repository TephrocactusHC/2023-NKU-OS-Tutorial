## 附录：执行`make qemu`的大致输出
```
$ make qemu
......
check_swap() succeeded!
++ setup timer interrupts
kernel_execve: pid = 2, name = "priority".
Breakpoint
set priority to 6
main: fork ok,now need to wait pids.
set priority to 5
set priority to 4
set priority to 3
set priority to 2
set priority to 1
child pid 7, acc 944000, time 2010 
child pid 6, acc 788000, time 2010 
child pid 5, acc 620000, time 2010 
child pid 4, acc 460000, time 2020 
child pid 3, acc 316000, time 2020 
main: pid 3, acc 316000, time 2020 
main: pid 4, acc 460000, time 2020 
main: pid 5, acc 620000, time 2030 
main: pid 6，acc 788000， time 2030 
main: pid 0, acc 944000, time 2030 
main: wait pids over
stride sched correctresult:1 1 2 2 3 
all user-mode processes have quit. 
init check memory pass.
kernel panic at kern/process proc.c:468:
    initproc exit.
```