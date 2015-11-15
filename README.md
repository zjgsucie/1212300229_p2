# 1212300229_p2
####周奎君，1212300229，Unix系统分析作业2：分析内核中cfs调度器相关代码。


首先来介绍一下cfs算法（完全公平调度算法）,为什么完全公平：

进程的运行时间计算公式为:
分配给进程的实际运行时间 = 调度周期 * 进程权重 / 所有进程权重之和   (公式1)

我们来看下从实际运行时间到vruntime的换算公式
vruntime = 实际运行时间 * 1024 / 进程权重 。 (公式2)

现在我们把公式2中的实际运行时间用公式1来替换，可以得到这么一个结果：
#####vruntime = (调度周期 * 进程权重 / 所有进程总权重) * 1024 / 进程权重 = 调度周期 * 1024 / 所有进程总权重 

没错，虽然进程的权重不同，但是它们的 vruntime增长速度应该是一样的 ，与权重无关。既然所有进程的vruntime增长速度宏观上看应该是同时推进的，那么就可以用这个vruntime来选择运行的进程，谁的vruntime值较小就说明它以前占用cpu的时间较短，受到了“不公平”对待，因此下一个运行进程就是它。这样既能公平选择进程，又能保证高优先级进程获得较多的运行时间。这就是CFS的主要思想。

现在开始分情景解析CFS。

###二、创建进程  

第一个情景选为进程创建时CFS相关变量的初始化。
我们知道，Linux创建进程使用fork或者clone或者vfork等系统调用，最终都会到do_fork。
如果没有设置CLONE_STOPPED，则会进入wake_up_new_task函数，我们看看这个函数的关键部分

```
/*  
 * wake_up_new_task - wake up a newly created task for the first time.  
 *  
 * This function will do some initial scheduler statistics housekeeping  
 * that must be done for every newly created context, then puts the task  
 * on the runqueue and wakes it.  
 */   
void  wake_up_new_task(struct  task_struct *p, unsigned long  clone_flags)  
{  
    .....  
    if  (!p->sched_class->task_new || !current->se.on_rq) {  
        activate_task(rq, p, 0);  
    } else  {  
        /*  
         * Let the scheduling class do new task startup  
         * management (if any):  
         */   
        p->sched_class->task_new(rq, p);  
        inc_nr_running(rq);  
    }  
    check_preempt_curr(rq, p, 0);  
    .....  
}  
```

p->sched_class->task_new对应的函数是task_new_fair:

```
/*  
 * Share the fairness runtime between parent and child, thus the  
 * total amount of pressure for CPU stays equal - new tasks  
 * get a chance to run but frequent forkers are not allowed to  
 * monopolize the CPU. Note: the parent runqueue is locked,  
 * the child is not running yet.  
 */   
static  void  task_new_fair(struct  rq *rq, struct  task_struct *p)  
{  
    struct  cfs_rq *cfs_rq = task_cfs_rq(p);  
    struct  sched_entity *se = &p->se, *curr = cfs_rq->curr;  
    int  this_cpu = smp_processor_id();  
    sched_info_queued(p);  
    update_curr(cfs_rq);  
    place_entity(cfs_rq, se, 1);  
    /* 'curr' will be NULL if the child belongs to a different group */   
    if  (sysctl_sched_child_runs_first && this_cpu == task_cpu(p) &&  
            curr && curr->vruntime < se->vruntime) {  
        /*  
         * Upon rescheduling, sched_class::put_prev_task() will place  
         * 'current' within the tree based on its new key value.  
         */   
        swap(curr->vruntime, se->vruntime);  
        resched_task(rq->curr);  
    }  
    enqueue_task_fair(rq, p, 0);  
}  
```

 这里有两个重要的函数，update_curr，place_entity。
其中update_curr在这里可以忽略，它是更新进程的一些随时间变化的信息，我们放到后面再看，
place_entity是更新新进程的vruntime值，以便把他插入红黑树。
新进程的vruntime确定之后有一个判断，满足以下几个条件时，交换父子进程的vruntime：
1.sysctl设置了子进程优先运行
2.fork出的子进程与父进程在同一个cpu上
3.父进程不为空（这个条件为什么会发生暂不明白，难道是fork第一个进程的时候？）
4.父进程的vruntime小于子进程的vruntime
几个条件都还比较好理解，说下第四个，因为CFS总是选择vruntime最小的进程运行，
因此必须保证子进程vruntime比父进程小，作者没有直接把子进程的vruntime设置为较小的值，
而是采用交换的方法，可以防止通过fork新进程来大量占用cpu时间，马上还要讲到。
最后，调用enqueue_task_fair将新进程插入CFS红黑树中

下面我们看下place_entity是怎么计算新进程的vruntime的。

```
static  void   
place_entity(struct  cfs_rq *cfs_rq, struct  sched_entity *se, int  initial)  
{  
    u64 vruntime = cfs_rq->min_vruntime;  
    /*  
     * The 'current' period is already promised to the current tasks,  
     * however the extra weight of the new task will slow them down a  
     * little, place the new task so that it fits in the slot that  
     * stays open at the end.  
     */   
    if  (initial && sched_feat(START_DEBIT))  
        vruntime += sched_vslice(cfs_rq, se);  
    if  (!initial) {  
        //先不看这里，   
    }  
    se->vruntime = vruntime;  
}  
```

这里是计算进程的初始vruntime。
它以cfs队列的min_vruntime为基准，再加上进程在一次调度周期中所增长的vruntime。
这里并不是计算进程应该运行的时间，而是先把进程的已经运行时间设为一个较大的值，
但是该进程明明还没有运行过啊，为什么要这样做呢？
假设新进程都能获得最小的vruntime(min_vruntime)，那么新进程会第一个被调度运行，
这样程序员就能通过不断的fork新进程来让自己的程序一直占据CPU，这显然是不合理的，
这跟以前使用时间片的内核中父子进程要平分父进程的时间片是一个道理。
再解释下min_vruntime，这是每个cfs队列一个的变量，它一般小于等于所有就绪态进程
的最小vruntime，也有例外，比如对睡眠进程进行时间补偿会导致vruntime小于min_vruntime。
至于sched_vslice计算细节暂且不细看，大体上说就是把概述中给出的两个公式结合起来如下：
sched_vslice = (调度周期 * 进程权重 / 所有进程总权重) * NICE_0_LOAD / 进程权重
也就是算出进程应分配的实际cpu时间，再把它转化为vruntime。
把这个vruntime加在进程上之后，就相当于认为新进程在这一轮调度中已经运行过了。

好了，到这里又可以回到wake_up_new_task(希望你还没晕，能回溯回去:-))，
看看check_preempt_curr(rq, p, 0);这个函数就直接调用了check_preempt_wakeup
```
static  void  check_preempt_wakeup(struct  rq *rq, struct  task_struct *p, int  sync)  
{  
    struct  task_struct *curr = rq->curr;  
    struct  sched_entity *se = &curr->se, *pse = &p->se; //se是当前进程，pse是新进程   
    /*  
     * Only set the backward buddy when the current task is still on the  
     * rq. This can happen when a wakeup gets interleaved with schedule on  
     * the ->pre_schedule() or idle_balance() point, either of which can  
     * drop the rq lock.  
     *  
     * Also, during early boot the idle thread is in the fair class, for  
     * obvious reasons its a bad idea to schedule back to the idle thread.  
     */   
    if  (sched_feat(LAST_BUDDY) && likely(se->on_rq && curr != rq->idle))  
        set_last_buddy(se);  
    set_next_buddy(pse);  
    while  (se) {  
        if  (wakeup_preempt_entity(se, pse) == 1) {  
            resched_task(curr);  
            break ;  
        }  
        se = parent_entity(se);  
        pse = parent_entity(pse);  
    }  
}  
```
首先对于last和next两个字段给予说明。
如果这两个字段不为NULL，那么last指向最近被调度出去的进程，next指向被调度上cpu的进程。
例如A正在运行，被B抢占，那么last指向A，next指向B。这两个指针有什么用呢?
当CFS在调度点选择下一个运行进程时，会优先照顾这两个进程，我们后面会看到，这里只要记住。<
这两个指针只使用一次，就是在上面这个函数退出后，返回用户空间时会触发schedule，
在那里选择下一个调度进程时会优先选择next，次优先选择last，选择完后，就会清空这两个指针。
这样设计的原因是，在上面的函数中检测结果是可以抢占并不代表已经抢占，而只是设置了调度标志，
在最后触发schedule时抢占进程B并不一定是最终被调度的进程(为什么？因为我们判断能否抢占
的根据是抢占进程B比运行进程A的vruntime小，但红黑树中可能有比抢占进程B的vruntime更小的进程C，
这样在调度时就会选中vruntime最小的C，而不是抢占进程B)，但是我们当然希望优先调度B，
因为我们就是为了运行B才设置了调度标志，所以这里用一个next指针指向B，以便给他个后门走，
如果B实在不争气，vruntime太大，就还是继续运行被抢占进程A比较合理，因此last指向被抢占进程，
这是一个比next小一点的后门，如果next走后门失败，就让被抢占进程A也走一次后门，
如果被抢占进程A也不争气，vruntime也太大，只好从红黑树中挑一个vruntime最小的了。
不管它们走后门是否成功，一旦选出下一个进程，就立刻清空这两个指针，不能老开着这个后门吧。
需要注意的是，schedule中清空这两个指针只在2.6.29及之后的内核才有，之前的内核没有那句话。
然后调用wakeup_preempt_entity检测是否满足抢占条件，如果满足（返回值为1）
则对当前进程设置TIF_NEED_RESCHED标志，在退出系统调用时会触发schedule函数进行进程切换,

###三、唤醒进程
```
/***  
 * try_to_wake_up - wake up a thread  
 * @p: the to-be-woken-up thread  
 * @state: the mask of task states that can be woken  
 * @sync: do a synchronous wakeup?  
 *  
 * Put it on the run-queue if it's not already there. The "current"  
 * thread is always on the run-queue (except when the actual  
 * re-schedule is in progress), and as such you're allowed to do  
 * the simpler "current->state = TASK_RUNNING" to mark yourself  
 * runnable without the overhead of this.  
 *  
 * returns failure only if the task is already active.  
 */   
static  int  try_to_wake_up(struct  task_struct *p, unsigned int  state, int  sync)  
{  
    int  cpu, orig_cpu, this_cpu, success = 0;  
    unsigned long  flags;  
    struct  rq *rq;  
    rq = task_rq_lock(p, &flags);  
    if  (p->se.on_rq)  
        goto  out_running;  
    update_rq_clock(rq);  
    activate_task(rq, p, 1);  
    success = 1;  
out_running:  
    check_preempt_curr(rq, p, sync);  
    p->state = TASK_RUNNING;  
out:  
    current->se.last_wakeup = current->se.sum_exec_runtime;  
    task_rq_unlock(rq, &flags);  
    return  success;  
} 
```
update_rq_clock就是更新cfs_rq的时钟，保持与系统时间同步。
重点是activate_task，它将进程加入红黑树并且对vruntime做一些调整，
然后用check_preempt_curr检查是否构成抢占条件，如果可以抢占则设置TIF_NEED_RESCHED标识。
因为check_preempt_curr讲过了，我们只顺着下面的顺序走一遍
   activate_task
-->enqueue_task
-->enqueue_task_fair
-->enqueue_entity
-->place_entity
###四、进程调度
```
/*  
 * schedule() is the main scheduler function.  
 */   
asmlinkage void  __sched schedule(void )  
{  
    struct  task_struct *prev, *next;  
    unsigned long  *switch_count;  
    struct  rq *rq;  
    int  cpu;  
need_resched:  
    preempt_disable(); //在这里面被抢占可能出现问题，先禁止它！   
    cpu = smp_processor_id();  
    rq = cpu_rq(cpu);  
    rcu_qsctr_inc(cpu);  
    prev = rq->curr;  
    switch_count = &prev->nivcsw;  
    release_kernel_lock(prev);  
need_resched_nonpreemptible:  
    spin_lock_irq(&rq->lock);  
    update_rq_clock(rq);  
    clear_tsk_need_resched(prev); //清除需要调度的位   
    //state==0是TASK_RUNNING，不等于0就是准备睡眠，正常情况下应该将它移出运行队列   
    //但是还要检查下是否有信号过来，如果有信号并且进程处于可中断睡眠就唤醒它   
    //注意对于需要睡眠的进程，这里调用deactive_task将其移出队列并且on_rq也被清零   
    //这个deactivate_task函数就不看了，很简单   
    if  (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {  
        if  (unlikely(signal_pending_state(prev->state, prev)))  
            prev->state = TASK_RUNNING;  
        else   
            deactivate_task(rq, prev, 1);  
        switch_count = &prev->nvcsw;  
    }  
    if  (unlikely(!rq->nr_running))  
        idle_balance(cpu, rq);  
    //这两个函数都是重点，我们下面分析   
    prev->sched_class->put_prev_task(rq, prev);  
    next = pick_next_task(rq, prev);  
    if  (likely(prev != next)) {  
        sched_info_switch(prev, next);  
        rq->nr_switches++;  
        rq->curr = next;  
        ++*switch_count;  
        //完成进程切换，不讲了，跟CFS没关系   
        context_switch(rq, prev, next); /* unlocks the rq */   
        /*  
         * the context switch might have flipped the stack from under  
         * us, hence refresh the local variables.  
         */   
        cpu = smp_processor_id();  
        rq = cpu_rq(cpu);  
    } else   
        spin_unlock_irq(&rq->lock);  
    if  (unlikely(reacquire_kernel_lock(current) < 0))  
        goto  need_resched_nonpreemptible;  
    preempt_enable_no_resched();  
    //这里新进程也可能有TIF_NEED_RESCHED标志，如果新进程也需要调度则再调度一次   
    if  (unlikely(test_thread_flag(TIF_NEED_RESCHED)))  
        goto  need_resched;  
}  
```
首先看put_prev_task，它等于put_prev_task_fair，后者基本上就是直接调用put_prev_entity
```
static  void  put_prev_entity(struct  cfs_rq *cfs_rq, struct  sched_entity *prev)  
{  
    /*  
     * If still on the runqueue then deactivate_task()  
     * was not called and update_curr() has to be done:  
     */   
    //记得这里的on_rq吗？在schedule函数中如果进程状态不是TASK_RUNNING，   
    //那么会调用deactivate_task将prev移出运行队列，on_rq清零。因此这里也是只有当   
    //prev进程仍然在运行态的时候才需要更新vruntime等信息。   
    //如果prev进程因为被抢占或者因为时间到了而被调度出去则on_rq仍然为1   
    if  (prev->on_rq)  
        update_curr(cfs_rq);  
    check_spread(cfs_rq, prev);  
    //这里也一样，只有当prev进程仍在运行状态的时候才需要更新vruntime信息   
    //实际上正在cpu上运行的进程是不在红黑树中的，只有在等待CPU的进程才在红黑树   
    //因此这里将调度出的进程重新加入红黑树。on_rq并不代表在红黑树中，而是代表在运行状态   
    if  (prev->on_rq) {  
        update_stats_wait_start(cfs_rq, prev);  
        /* Put 'current' back into the tree. */   
        //这个函数也不跟进去了，就是把进程以(vruntime-min_vruntime)为key加入到红黑树中   
        __enqueue_entity(cfs_rq, prev);  
    }  
    //没有当前进程了，这个当前进程将在pick_next_task中更新   
    cfs_rq->curr = NULL;  
}  
```
再回到schedule中看看pick_next_task函数，基本上也就是直接调用pick_next_task_fair
```
static  struct  task_struct *pick_next_task_fair(struct  rq *rq)  
{  
    struct  task_struct *p;  
    struct  cfs_rq *cfs_rq = &rq->cfs;  
    struct  sched_entity *se;  
    if  (unlikely(!cfs_rq->nr_running))  
        return  NULL;  
    do  {  
        //这两个函数是重点，选择下一个要执行的任务   
        se = pick_next_entity(cfs_rq);  
        set_next_entity(cfs_rq, se);  
        cfs_rq = group_cfs_rq(se);  
    } while  (cfs_rq);  
    p = task_of(se);  
    hrtick_start_fair(rq, p);  
    return  p;  
}  
```
主要看下pick_next_entity和set_next_entity
```
static  struct  sched_entity *pick_next_entity(struct  cfs_rq *cfs_rq)  
{  
    //__pick_next_entity就是直接选择红黑树缓存的最左结点，也就是vruntime最小的结点   
    struct  sched_entity *se = __pick_next_entity(cfs_rq);  
    //下面的wakeup_preempt_entity已经讲过，忘记的同学可以到上面看下   
    //这里就是前面所说的优先照顾next和last进程，只有当__pick_next_entity选出来的进程   
    //的vruntime比next和last都小超过调度粒度时才轮到它运行，否则就是next或者last   
    if  (cfs_rq->next && wakeup_preempt_entity(cfs_rq->next, se) < 1)  
        return  cfs_rq->next;  
    if  (cfs_rq->last && wakeup_preempt_entity(cfs_rq->last, se) < 1)  
        return  cfs_rq->last;  
    return  se;  
}  
static  void   
set_next_entity(struct  cfs_rq *cfs_rq, struct  sched_entity *se)  
{  
    /* 'current' is not kept within the tree. */   
    //这里什么情况下条件会为假？我以为刚唤醒的进程可能不在rq上   
    //但是回到上面去看了下，唤醒的进程也通过activate_task将on_rq置1了   
    //新创建的进程on_rq也被置1，这里什么情况会为假，想不出来   
    //这里我也测试了一下，在条件为真假的路径上各设置了一个计数器   
    //当条件为真经过了将近五十万次的时候条件为假仅有一次，   
    //所以我们可以认为基本上都会直接进入if语句块执行   
    if  (se->on_rq) {  
        /*这里注释是不是写错了？dequeued写成了enqueued?  
         * Any task has to be enqueued before it get to execute on  
         * a CPU. So account for the time it spent waiting on the  
         * runqueue.  
         */   
        update_stats_wait_end(cfs_rq, se);  
        //就是把结点从红黑树上取下来。前面说过，占用CPU的进程不在红黑树上   
        __dequeue_entity(cfs_rq, se);  
    }  
    update_stats_curr_start(cfs_rq, se);  
    cfs_rq->curr = se;  //OK，在put_prev_entity中清空的curr在这里被更新   
    //将进程运行总时间保存到prev_..中，这样进程本次调度的运行时间可以用下面公式计算：   
    //进程本次运行已占用CPU时间 =  sum_exec_runtime - prev_sum_exec_runtime   
    //这里sum_exec_runtime会在每次时钟tick中更新   
    se->prev_sum_exec_runtime = se->sum_exec_runtime;  
}  
```
到此schedule函数也讲完了。
关于dequeue_task，dequeue_entity和__dequeue_entity三者区别
前两者差不太多，不同的那一部分我也没看明白。。。主要是它们都会将on_rq清零，
我觉得是当进程要离开TASK_RUNNING状态时调用，这两个函数可以将进程取下运行队列。
而__dequeue_entity不会将on_rq清零，只是将进程从红黑树上取下，
我觉得一般用在进程将获得CPU的情况，这时需要将它从红黑树取下，但是还要保留在rq上。

###参考博客：[http://blog.csdn.net/peimichael/article/details/5218335](http://blog.csdn.net/peimichael/article/details/5218335)
