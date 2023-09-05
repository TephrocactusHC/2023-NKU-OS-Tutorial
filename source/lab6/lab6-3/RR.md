### RR调度算法实现

时间片轮转调度(Round-Robin Scheduling)算法非常简单。它为每一个进程维护了一个最大运行时间片。当一个进程运行够了其最大运行时间片那么长的时间后，调度器会把它标记为需要调度，并且把它的进程控制块放在队尾，重置其时间片。这种调度算法保证了公平性，每个进程都有均等的机会使用CPU，但是没有区分不同进程的优先级（这个也就是在Stride算法中需要考虑的问题）。下面我们来实现以下时间片轮转算法相对应的调度器接口吧！

首先是`enqueue`操作。RR算法直接把需要入队的进程放在调度队列的尾端，并且如果这个进程的剩余时间片为0（刚刚用完时间片被收回CPU），则需要把它的剩余时间片设为最大时间片。具体的实现如下：

```c
static void
RR_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    assert(list_empty(&(proc->run_link)));
    list_add_before(&(rq->run_list), &(proc->run_link));
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;
    }
    proc->rq = rq;
    rq->proc_num ++;
}
```

`dequeue`操作非常普通，将相应的项从队列中删除即可：

```c
static void
RR_dequeue(struct run_queue *rq, struct proc_struct *proc) {
    assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
    list_del_init(&(proc->run_link));
    rq->proc_num --;
}
```

`pick_next`选取队列头的表项，用`le2proc`函数获得对应的进程控制块，返回：

```c
static struct proc_struct *
RR_pick_next(struct run_queue *rq) {
    list_entry_t *le = list_next(&(rq->run_list));
    if (le != &(rq->run_list)) {
        return le2proc(le, run_link);
    }
    return NULL;
}
```

`proc_tick`函数在每一次时钟中断调用。在这里，我们需要对当前正在运行的进程的剩余时间片减一。如果在减一后，其剩余时间片为0，那么我们就把这个进程标记为“需要调度”，这样在中断处理完之后内核判断进程是否需要调度的时候就会把它进行调度：

```c
static void
RR_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
    if (proc->time_slice > 0) {
        proc->time_slice --;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
}
```

至此我们就实现完了和时间片轮转算法相关的所有重要接口。类似于RR算法，我们也可以参照这个方法实现自己的调度算法。本次实验中需要同学们自己实现Stride调度算法。