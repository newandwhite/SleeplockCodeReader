# include/linux/semaphore.h

### 信号量数据结构

信号量是一种经典的同步机制，用于控制并发访问的同步机制。在Linux内核中，信号量通过struct semaphore结构体来实现，它的主要特点是：

* 支持多进程或多线程之间的同步。
* 可以用于保护共享资源的访问。

当某个进程或线程需要访问共享资源时，它会先尝试获取信号量。如果信号量的计数器大于0，表示有可用的资源，该进程或线程就可以继续执行，并将信号量计数器减1；否则，它会被阻塞并加入等待队列，直到有其他进程或线程释放了资源并增加了信号量计数器。  
下方是信号量的结构定义：

```
struct semaphore {
    raw_spinlock_t        lock;
    unsigned int        count;
    struct list_head    wait_list;
};
```

* lock: 自旋锁变量，用于保护count和wait\_list
* count: 是表示当前信号量的计数器值，信号量的计数器表示可用资源的数量。当计数器大于零时，表示有可用资源，可以继续访问共享资源；当计数器等于零时，表示资源已经被占用，需要等待其他进程或线程释放资源
* wait\_list: 等待本信号量的所有睡眠进程。这是一个链表的头结构，用于维护等待该信号量的进程或线程的队列。当一个进程或线程等待信号量时，它会将自己的控制数据结构（如进程控制块或线程控制块）添加到 wait\_list 链表中。当信号量的计数器为零时，正在等待的进程或线程会被放置在该链表中，直到有其他进程或线程释放资源。

### 信号量初始化

```
#define __SEMAPHORE_INITIALIZER(name, n)                \
{                                    \
    .lock        = __RAW_SPIN_LOCK_UNLOCKED((name).lock),    \
    .count        = n,                        \
    .wait_list    = LIST_HEAD_INIT((name).wait_list),        \
}

#define DEFINE_SEMAPHORE(_name, _n)    \
    struct semaphore _name = __SEMAPHORE_INITIALIZER(_name, _n)

static inline void sema_init(struct semaphore *sem, int val)
{
    static struct lock_class_key __key;
    *sem = (struct semaphore) __SEMAPHORE_INITIALIZER(*sem, val);
    lockdep_init_map(&sem->lock.dep_map, "semaphore->lock", &__key, 0);
}
```

初始化一个新的信号量，并设置初始计数器值。  
支持两种初始化方式，一种是宏定义的方式声明，一种是显式调用。

---

# kernel/locking/semaphore.c

### 各类down操作与用途

```
void __sched down(struct semaphore *sem);
int __sched down_interruptible(struct semaphore *sem);
int __sched down_killable(struct semaphore *sem);
int __sched down_timeout(struct semaphore *sem, long timeout);
int __sched down_trylock(struct semaphore *sem);
```

* down：获取信号量sem\(信号量的值减 1\)，当信号量的值非负时，可以立即获取信号量，否则进程休眠。该函数用于获得信号量，它会导致睡眠，因此不能在中断上下文使用该函数。该函数将把sem的值减1，如果信号量sem的值，就直接返回，否则调用者将被挂起，直到别的任务释放该信号量才能继续运行。
* down\_interruptible：同 down\(\)，区别是若获取不到进入 TASK\_INTERRUPTIBLE 状态，支持被信号唤醒。down不会被信号（signal）打断，但down\_interruptible能被信号打断，因此该函数有返回值来区分是正常返回还是被信号中断，如果返回0，表示获得信号量正常返回，如果被信号打断，返回-EINTR。
* down\_killable：同 down\(\)，区别是若获取不到进入 TASK\_KILLABLE\(TASK\_WAKEKILL \| TASK\_UNINTERRUPTIBLE\)状态，表示只可被 SIGKILL 信号唤醒。睡眠的进程可以因为受到致命信号而被唤醒，中断获取信号量的操作。
* down\_timeout：同down\(\)，区别是若获取不到不会一直等待，在jiffies个时钟周期后如果还没有获取信号量，则超时返回，返回0表示成功获取信号量，返回负值表示超时。到期后自动唤醒。
* down\_trylock：尝试获取信号量，不会阻塞，返回1成功，0失败。

#### down

```
void __sched down(struct semaphore *sem)
{
    unsigned long flags;

    might_sleep();
    raw_spin_lock_irqsave(&sem->lock, flags);
    if (likely(sem->count > 0))
        sem->count--;
    else
        __down(sem);
    raw_spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(down);
```

函数分析：函数首先通过自旋锁raw\_spin\_lock\_irqsave的调用来保证对sem-&gt;count操作的原子性。如果count&gt;0，表示当前进程可以获得信号量，将count的值减1然后退出。如果count不大于0，表明当前进程无法获取信号量，则调用\_\_down\(sem\)函数，后者会继续调用\_\_down\_common。

#### down\_interruptible

```
int __sched down_interruptible(struct semaphore *sem)
{
    unsigned long flags;
    int result = 0;

    might_sleep();
    raw_spin_lock_irqsave(&sem->lock, flags);
    if (likely(sem->count > 0))
        sem->count--;
    else
        result = __down_interruptible(sem);
    raw_spin_unlock_irqrestore(&sem->lock, flags);

    return result;
}
EXPORT_SYMBOL(down_interruptible);
```

函数分析：与down函数相同，区别在于count不大于0的情况下调用\_\_down\_interruptible，后者会继续调用\_\_down\_common。

```
static noinline int __sched __down_interruptible(struct semaphore *sem)
{
    return __down_common(sem, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}
```

#### down\_killable

```
int __sched down_killable(struct semaphore *sem)
{
    unsigned long flags;
    int result = 0;

    might_sleep();
    raw_spin_lock_irqsave(&sem->lock, flags);
    if (likely(sem->count > 0))
        sem->count--;
    else
        result = __down_killable(sem);
    raw_spin_unlock_irqrestore(&sem->lock, flags);

    return result;
}
EXPORT_SYMBOL(down_killable);
```

函数分析：与down函数相同，区别在于count不大于0的情况下调用\_\_down\_killable，后者会继续调用\_\_down\_common。

```
static noinline int __sched __down_killable(struct semaphore *sem)
{
    return __down_common(sem, TASK_KILLABLE, MAX_SCHEDULE_TIMEOUT);
}
```

#### down\_timeout

```
int __sched down_timeout(struct semaphore *sem, long timeout)
{
    unsigned long flags;
    int result = 0;

    might_sleep();
    raw_spin_lock_irqsave(&sem->lock, flags);
    if (likely(sem->count > 0))
        sem->count--;
    else
        result = __down_timeout(sem, timeout);
    raw_spin_unlock_irqrestore(&sem->lock, flags);

    return result;
}
EXPORT_SYMBOL(down_timeout);
```

函数分析：与down函数相同，区别在于count不大于0的情况下调用\_\_down\_timeout，后者会继续调用\_\_down\_common。

```
static noinline int __sched __down_timeout(struct semaphore *sem, long timeout)
{
    return __down_common(sem, TASK_UNINTERRUPTIBLE, timeout);
}
```

#### down\_trylock

```
int __sched down_trylock(struct semaphore *sem)
{
    unsigned long flags;
    int count;

    raw_spin_lock_irqsave(&sem->lock, flags);
    count = sem->count - 1;
    if (likely(count >= 0))
        sem->count = count;
    raw_spin_unlock_irqrestore(&sem->lock, flags);

    return (count < 0);
}
EXPORT_SYMBOL(down_trylock);
```

该函数试着获得信号量sem，如果能够立刻获得，它就获得该信号量并返回0，否则，表示不能获得信号量sem，返回值为非0值。因此，它不会导致调用者睡眠，可以在中断上下文使用。

#### \_\_down\_common函数

在不同down操作下对该函数传入了不同的参数，表示task不同的状态。这些状态的宏定义在include/linux/sched.h中。举例如下：

```
#define TASK_RUNNING            0x00000000
#define TASK_INTERRUPTIBLE        0x00000001
#define TASK_UNINTERRUPTIBLE        0x00000002
#define TASK_WAKEKILL            0x00000100
#define TASK_KILLABLE            (TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
```

以下是\_\_down\_common函数本身。

> 有一个问题就是我找不到trace\_contention\_begin\(\)和trace\_contention\_end\(\)的定义，虽然应该不影响整体分析。

```
static inline int __sched __down_common(struct semaphore *sem, long state,
                    long timeout)
{
    int ret;

    trace_contention_begin(sem, 0);
    ret = ___down_common(sem, state, timeout);
    trace_contention_end(sem, ret);

    return ret;
}
```

在\_\_down\_common函数里调用\_\_\_down\_common\(\)函数。

```
static inline int __sched ___down_common(struct semaphore *sem, long state,
                                long timeout)
{
    struct semaphore_waiter waiter; //创建一个名为 waiter 的信号量等待者结构体

    list_add_tail(&waiter.list, &sem->wait_list); //将 waiter 结构体添加到信号量的等待队列末尾。这将把当前任务添加到等待队列中，表示它正在等待获取信号量
    waiter.task = current; //将task关联到当前正在执行的任务（进程或线程），表示该等待者对应于当前任务。
    waiter.up = false; //将waiter结构体的up字段设置为false，表示该等待者尚未获得信号量。

    //开始一个无限循环，用于等待获得信号量
    for (;;) {
        //检查当前任务是否有待处理的信号（如中断信号或终止信号）。如果有，则跳转到标签 interrupted，表示获取操作被中断
        if (signal_pending_state(state, current))
            goto interrupted;
        //检查等待超时时间是否小于等于零。如果是，则跳转到标签 timed_out，表示等待超时
        if (unlikely(timeout <= 0))
            goto timed_out;
        __set_current_state(state); //将当前任务的状态设置为 state，通常是一个阻塞状态（例如，TASK_INTERRUPTIBLE 或 TASK_UNINTERRUPTIBLE），表示任务正在等待
        raw_spin_unlock_irq(&sem->lock); //解锁信号量的自旋锁，允许其他任务访问信号量
        timeout = schedule_timeout(timeout); //使当前任务进入睡眠状态，并等待指定的超时时间或直到被唤醒
        raw_spin_lock_irq(&sem->lock); //当被唤醒后，重新获取信号量的自旋锁
        //检查等待者的 up 字段是否被设置为 true。如果是，表示该等待者已经成功获取信号量（通过其他任务释放资源），函数返回 0，表示获取成功
        if (waiter.up)
            return 0;
    }

 timed_out:
    list_del(&waiter.list); //从信号量的等待队列中删除 waiter 结构体
    return -ETIME; //返回一个代表等待超时的错误代码-ETIME，表示等待超时

 interrupted:
    list_del(&waiter.list);
    return -EINTR; //返回一个代表等待被中断的错误代码-EINTR，表示获取过程被中断
}
```

在\_\_down\_common函数数执行了以下操作。  
1. 将当前进程放到信号量成员变量wait\_list所管理的队列中。  
2. 在一个for循环中把当前的进程状态这是为TASK\_INTERRUPTIBLE,在调用schedule\_timeout使当前进程进入睡眠状态，函数将停留在schedule\_timeout调用上，知道再次被调度执行。  
3. 当该进程再一次被调度时，按原因执行相应的操作：如果waiter.up不为0说明进程被该信号量的up操作所唤醒，进程可以获得信号量。如果进程是因为被用户空间的信号所中断或超时信号所引起的唤醒，则返回相应的错误代码。

其中涉及到的结构体为：

```
struct semaphore_waiter {
    struct list_head list;
    struct task_struct *task;
    bool up;
};
```

### up操作

仅有一个函数，该函数释放信号量sem，即把sem的值加1，如果sem的值为非正数，表明有任务等待该信号量，因此唤醒这些等待者。

```
void __sched up(struct semaphore *sem)
{
    unsigned long flags;

    raw_spin_lock_irqsave(&sem->lock, flags);
    if (likely(list_empty(&sem->wait_list)))
        sem->count++;
    else
        __up(sem);
    raw_spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(up);
```

up\(\)函数可以被任何上下文调用，包括从未调用down\(\)函数的任务。其参数是一个指向信号量的指针。当信号量的计数值被释放时，将会唤醒等待队列中的一个进程，使其继续执行。计数信号量的上升操作不会引起进程的睡眠，因此可以在任何上下文中调用。  
在函数内部，通过原子操作rawspin\_lock\_irqsave\(\)和raw\_spin\_unlock\_irqrestore\(\)保证了信号量的原子性和线程安全性。  
首先，通过list\_empty\(\)函数判断等待队列sem-&gt;wait\_list是否为空。如果等待队列为空，则将等待信号量那么就直接将count进行++，如果有那么调用\\_up\(sem\)函数进行处理，并将信号量的计数值保持不变。  
这个函数是一个非阻塞函数，不会引起进程的睡眠。如果等待队列中有等待的进程，则会唤醒其中的一个进程，使其继续执行；否则，计数值将会增加1。

\_\_up\(\)函数在下方列出。

```
static noinline void __sched __up(struct semaphore *sem)
{
    struct semaphore_waiter *waiter = list_first_entry(&sem->wait_list,
        struct semaphore_waiter, list); //从等待链表中取出第一个在等待的进程
    list_del(&waiter->list); //将它从等待链表中删除
    waiter->up = true; //标识已经获取到了信号量
    wake_up_process(waiter->task); //唤醒得到信号量的进程
}
```

### down与up操作的实现

即P操作和V操作（获取操作和释放操作）。在Linux内核中，它们通常被分别实现为up\(\)和down\_interruptible\(\)函数。

* P操作（获取操作）：使用down\_interruptible\(\)函数实现。它将一个二进制信号量的值减1，或者将一个计数信号量的值减n（n为参数）。如果信号量的值为0，则当前进程会被阻塞，并加入到该信号量的等待队列中。
* V操作（释放操作）：使用up\(\)函数实现。它将一个二进制信号量的值加1，或者将一个计数信号量的值加n（n为参数），并唤醒任何等待该信号量的进程。

详细实现见上方函数分析。

