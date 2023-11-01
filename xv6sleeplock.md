有时，xv6 需要长时间的持有一个锁，比如文件系统会在读写硬盘上的内容时持有锁，而这些操作往往要花费数十毫米。长时间持有一个 spinlock 会导致浪费，因为在 spin 期间 CPU 做不了任何事情。spinlock 的另一个缺陷是在持续期间内不能让出 CPU。我们想要做的就是如果一个因为一个锁在等待磁盘操作的花，一个进程可以使用 CPU。持有 spinlock 期间让出 CPU 是不合法的，因为如果其他进程获取这个锁会导致整个操作系统死锁。因此，我们需要一种锁，当执行 acquire 的时候让出 CPU。并在持有锁的时候允许让出 CPU。

xv6 提供了这样一种锁 sleep-locks 以在等待时让出 CPU。从上层来讲，一个 sleeplock 拥有一个让 spinlock 保护的字段 locked。acquiresleep 将调用 sleep 以让出 CPU，并释放 spinlock。结果就是当执行 acquiresleep 的时候，其他线程可以执行。

由于 sleep-locks 启用了中断，所以不能被用于中断处理中。因为acquiresleep 可能会让出 CPU，sleep-lock 不能在 spinlock 的临界区内使用。spin-locks 适用于短临界区，因为等待总会浪费 CPU。而 sleep-lock 在耗时较长的操作上表现很好。

# sleeplock.h
sleeplock.h定义了睡眠锁的结构包含记录是否已经被持有的locked和保护sleeplock的spinlock（保护对临界变量的访问），还有pid和锁名。
# sleeplock.c
### initsleeplock()
通过调用initlock()和赋值来对锁初始化。

### acquiresleep
取锁
```
void acquiresleep(struct sleeplock *lk)
{
    acquire(&lk->lk); // 优先级:->高于&
    while (lk->locked) { // 当锁已被其他进程取走
        sleep(lk, &lk->lk); // 休眠
    }
    lk->locked = 1; // 上锁
    lk->pid = myproc()->pid; // 取锁进程的pid
    release(&lk->lk);
}
```
acquiresleep()在获取sleeplock的过程中使用其中的spinlock保证函数的原子性。先获取对应的spinlock，然后判断要获取的锁是否已被持有，如果被持有则进入睡眠，当从睡眠醒来时，需要重新获取spinlock。如果可以获取锁，就改变locked和pid，然后释放spinlock。进入sleep函数时，会获取当前进程的lock，然后释放sleeplock的自旋锁，而sleep函数结束时会释放进程的锁，获取睡眠锁的锁。函数采用while语句，让每次进程被唤醒时都检查等待的条件是否满足，防止出现一个睡眠进程等待的锁没有空闲就被唤醒的情况。

### releasesleep
解锁
```
void releasesleep(struct sleeplock *lk)
{
    acquire(&lk->lk); //取自旋锁
    lk->locked = 0;
    lk->pid = 0;
    wakeup(lk); //唤醒
    release(&lk->lk);
}
```
releasesleep()同样使用自旋锁维护原子性，释放时把锁的pid和locked值设为0，然后使用wakeup函数，实际上是对设置进程状态p->state=RUNNABLE的包装。

> 为什么 sleeplock 还需要一个 spinlock 来保护？

如果不适用spinlock（自旋锁）保护，sleeplock未被持有时，两个进程相继申请锁，都无法上锁。当进程1释放锁时，进程2申请该锁，使阻塞队列进程和进程2同时持有该锁。


锁的顺序
如果一段代码要使用多个锁，那么必须要注意代码每次运行都要以相同的顺序获得锁，并以相反的顺序释放锁，否则就会发生死锁。假设某段代码中存在两条路径，路径1获得锁的顺序是 A、B，而路径2获得锁的顺序是 B、A。那么就可能1中持有A，2中持有B，二者互相等传无法继续运行，发生死锁。为了避免死锁，所有的代码路径获得锁的顺序必须相同。同样的，释放锁必须以获得的相反顺序运行。加锁的操作就像给代码加上花括号，内层对应内层，外层对应外层。
> 获取睡眠锁时，进程状态的转换过程 

![](/assets/sleeplock_state.png)

> 允许两个进程同时使用临界区的修改

* 睡眠锁中的 sleep() 和 wakeup() 属于互斥同步中的阻塞操作。
* 调用sem_init，传入信号量的指针，初始值设置为0，表明这是一个计数信号量counting semaphore
* 将原来acquire和release中的p->lock判断修改为：信号量的值小于2、信号量的值减1

> acquiresleep() 调用的 sleep() 函数中，为什么在调用 acquire(&p->lock) 后紧接着调用 release(lk) ？ 
acquire(&p->lock) 和 release(lk) 两行的函数调用能否互换？

调用进程从acquire(lk)开始就持有lk，直到在sleep中获取了p->lock之后才释放lk，保证了在这个过程中，不会有别的wakeup(chan)被调用。 当sleep获取了p->lock然后释放lk之后，其它的wakeup(chan)调用虽然可以开始执行，但是又会阻塞在获取p->lock上，等待直到sleep将调用进程挂起，因此不会有唤醒丢失wake loss（即一个正在睡眠的进程错过了唤醒执行的机会）的问题。 

