## Lec10-Locking

*2023-12-7*

#### **锁对性能的影响**

锁虽然保证了共享数据结构的一致性，但是也会限制多核性能

例如`kfree()`，两个CPU核同时要释放page，但是只有一个free list情况下两个操作只能串行(Serialization)，多核的优势就无从发挥，这称为锁的争夺 (Contention)

有一个解决办法是，每个CPU维护一个私有的free list，只有在自己的free list耗尽的情况下，才会去“偷”其他CPU的free list

这种私有化加速的思想和昨天看的`glibc`堆管理的**`tcache`**机制有异曲同工之妙。一个窗口排队太慢，那就多开几个窗口提供服务

课上还提到了`lock free program`，不知道是如何实现的

####  **锁的使用**

两种情况下需要使用锁

1. 一个变量可能同时被多个CPU访问修改
2. 一个不变量的维护涉及多个内存操作，需要保证这一连串内存操作的原子性

第一点不难理解，第二点的一个例子如课上提到的: `rename d1/a d2/b`，这里有两个操作，先删除d1下的inode，然后在d2下添加inode，如果分开执行，那么一小段时间内，在系统看来这个inode消失了

锁的粒度(granularity)会影响并行性能

`coarse-grained locking`：xv6的`kalloc.c`对一个只使用了一把锁`kmem`，粒度大，排队时间长

`fine-grained locking`：xv6的每个文件都有一把独立的锁，操作不同文件可以并行进行

####  **死锁与锁的顺序**

要避免死锁，就要事先规定好所有程序获取锁的顺序，但这会带来很多问题

1. 抽象泄露：两个模块间不应了解对方的实现细节，但为了`lock order`不得不了解
2. 有悖逻辑：模块M1调用M2，但`lock order`却要求先获取M2中的一把锁，再获取M1的
3. Sometimes the identities of locks aren't known in advance(**P61 没懂**)

粒度越细，锁就越多，就越可能造成死锁

#### **中断处理中锁的问题**

如果进程A获取了一个`spinlock`并在执行，突然产生一个中断，该中断处理函数也要同一个`spinlock`，那这个CPU就完蛋了，会永远在这个`spinlock`上转啊转，因为这个锁只能被进程A释放，而进程A执行又要等到中断处理函数返回，也就是说这个CPU死锁了

所以原则上来说，如果一个中断处理函数需要一个**`spinlock`**，那么**CPU必须把该中断关了**才能获取这个`spinlock`

Xv6的策略是在`acquire(&lock)`的开头做一个`push_off()`，这个函数做两件事

1. 关闭**当前CPU**的中断，如果这个是CPU获取的第一把锁，就保存之前的中断状态到`mycpu()->intena`
2. 增加`mycpu()->noff`，获取几把锁就嵌套几层

对应的，`release(&lock)`在末尾调用一次`pop_off()`

1. 减少`mycpu()->noff`，减少一层嵌套
2. 如果所有锁都释放完了，那么恢复CPU之前的中断状态

这样就保证了只要该CPU持有锁，那么它就不会被中断

####  **Sleep locks**

`spinlock`不会让出CPU资源，所以CPU会一直阻塞直到`acquire(&spinlock)`返回，可能非常消耗CPU资源

而且持有`spinlock`时，如果`yield`了，可能会造成类似中断问题的死锁局面，也违背了关中断原则

这时候就需要`sleeplock`了，`acquiresleep(&sleeplock)`如果等不到，就直接去睡觉了，让出当前CPU资源，因此`sleeplock`不会关闭中断。也正因如此，所以在`spinlock`的临界区中不能调用`sleeplock`，否则又可能死锁



***PS***：xv6的锁就两类，非常简单，也没有解决starvation, fairness等问题，毕竟“小而精”，涉及的内容不如上学期ICS讲到的有关OSTEP的部分



<hr/>

## Lec11-Thread switching

Xv6并没有实现POSIX标准的`pthread`库，一个用户进程只能有一个用户线程和一个内核线程，所以这里的上下文切换对于进程和线程并没有那么清晰的界限

#### **上下文切换**

Xv6抽象了一层“调度器线程”，即每一个core都有一个自己的无限循环的`scheduler()`作为最顶层的管理者

从一个用户线程切换到另一个用户线程都要经历 **用户线程->内核线程->调度器线程->内核线程->用户线程**

切换的核心代码在于**`swtch`**`(swtch.S)`，必须要用汇编是因为这涉及到寄存器操作（特别是`sp,ra`这些），而C不能直接访问寄存器，就和trap机制一样，`trampoline`也是由汇编编写

**`swtch(&old, &new)`**把old线程的所有`Callee-saved`寄存器和**`ra,sp`**保存到当前线程的**`context`**中，然后再加载new线程的context。

关键点来了：因为加载了new线程的`ra,sp`寄存器，当前CPU就无缝切换到了new线程在很早之前对`swtch`的调用返回位置；而对于old线程来说，**整个世界定格在了内存中**，它对`swtch`的调用静止了，等待着调度器再次`swtch`到它，就如同new线程一样，时间才会继续流动

####  **线程调度**

在进入调度器前，进程要做一系列动作来保证invariants，因为`swtch`会破坏context的一致性

1. 获取进程锁`(p->lock)`
2. 释放其他锁`(mycpu()->noff == 1)`
3. 修改运行状态为RUNNABLE
4. 关闭CPU中断

确保这些条件满足后，才会进入调用`swtch()`进入调度器，否则kernel panic

不难注意到，这里的进程锁在`swtch`前后被跨线程交接了，内核线程获取的进程锁由调度器线程释放，反之亦然，这为了防止多个CPU抢到同一个RUNNABLE线程

 观察调度时pc的运行顺序，发现线程切换点总是在`sched()`和`scheduler()`，可以认为二者互相等待对方“返回”，这是一种非抢占式的调度，可以认为是[**协程**](https://zhuanlan.zhihu.com/p/446999661)

- **线程第一次调度**

子进程在第一次被fork时，它的状态会被设为RUNNABLE，但由于它自己并没有调用`sched()`，所以需要“伪造”一个context，让调度器在第一次调度到它时能正确返回到fork()后的位置。

```c
  /* allocproc:143 (proc.c) */
  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;
```


这样在第一次调度后，子进程会从`forkret()`开始执行，`forkret()`的任务仅仅是释放了锁，然后调用`usertrapret()`，跳到子进程的用户空间

####  `mycpu() & myproc()`

内核代码中很多地方都要调用`myproc()`来获取当前CPU上运行的进程的结构指针，在多核机器上，就需要保存每颗CPU上运行的进程信息，xv6定义了如下结构体

```c
// Per-CPU state.
struct cpu {
  struct proc *proc;          // The process running on this cpu, or null.
  struct context context;     // swtch() here to enter scheduler().
  int noff;                   // Depth of push_off() nesting.
  int intena;                 // Were interrupts enabled before push_off()?
};

extern struct cpu cpus[NCPU];
```

这个`cpus[NCPU]`数组的索引就要利用到RISC-V的特性——每颗CPU都有一个编号**`hartid`** (Hardware thread ID，xv6在内核态中确保这个编号保存在**`tp`**寄存器中，因此就能通过这个寄存器来索引当前CPU，获取当前进程

需要注意的是`cpuid()`和`mycpu()`返回值是很脆弱的，如果在读取时发生了定时器中断，那么下一次该进程可能被另一个CPU认领了，而返回值依然是上一个CPU的`tp`，就会出现问题

所以为了保持一致性，调用这两个函数时必须要关中断

但`myproc()`不用，因为就算发生了调度，它返回的还是那个进程指针

<hr/>

## Lec13-Sleep and wakeup

*2023-12-15*

####  Lost wakeups

如果P操作在检查锁条件时没有维护锁的一致性，就可能会丢失来自V操作的wakeup

```c
void V(struct semaphore *s) {
    acquire(&s->lock);
    s->count += 1;
    wakeup(s);
    release(&s->lock);
}

void P(struct semphore *s) {
	while(s->count == 0)
		/* RIGHT HERE!! */
		sleep(s);
	acquire(&s->lock);
    s->count -= 1;
    release(&s->lock);
}
```

如果A线程在P检查完s->count发现等于零后，发生了线程切换，B线程的一个V操作wakeup了所有睡在信号量s上的线程，但这时A还没go to sleep，切换回A后，A才开始sleep，但那个wakeup却已经一去不复返了，A高概率陷入永眠，直到有下一个V操作来唤醒它。

这个bug本质是条件变量s的一致性被破坏导致的，wakeup超前于sleep，这是我们不希望看到的。因此朴素的想法是在线程sleep之后再释放`s->lk`，但在xv6中，sleep时会获取`p->lock`，然后就可以释放`s->lk`·了，就算发生了线程切换，wakeup在查询进程状态前也要等待`p->lock`释放，使得wakeup不会丢失

这种锁类似Unix中的Conditional variables

####  Pipes

Xv6中的管道采用生产者-消费者模型，以内核中的一块空间为缓冲区，`pipewrite`在缓冲区写满时唤醒`piperead`，反之`piperead`在缓冲区空或者读完n个字符后唤醒`pipewrite`，如果管道的任何一端引用数为零，那么管道关闭

####  exit

进程退出后，要清除状态，释放栈、用户内存、pagetable、trapframe等资源，但是进程不能只靠自己释放全部资源，因为这些资源是进程运行所必需的，因此需要有其他进程来帮助其彻底释放。

Xv6中，进程在自己调用`exit()`后，会关闭打开的文件描述符，将自己的所有子进程reparent给`init`进程，唤醒正在wait的父进程，再将自己的状态设为ZOMBIE，最后调用`sched()`永久让出CPU资源

####  wait

wait会找到自己的子进程，然后判断是否是ZOMBIE，如果是，就调用`freeproc()`来彻底释放子进程资源，使其REUSEABLE。如果没有子进程退出，那么就会进入sleep

####  kill

kill不能直接暴力释放目标进程的所有资源，因为很可能目标进程正处于critical section，一旦直接释放，很可能造成死锁。因此kill只是简单地设置`p->killed = 1`，并唤醒目标进程。

有些进程在被唤醒后可能不会检查`p->killed`，因为其可能正在执行一系列原子操作，比如磁盘操作，需要在所有操作完成后，保证文件系统正确性才能退出

#### Real world

- 线程调度

Xv6中的调度策略只是简单的Round robin。考虑到公平性和吞吐率，一般操作系统中会采用带优先级的调度策略，像ics2里学的MLFQ，但这会带来问题，比如***优先级反转(priority inversion)***和***争夺(convoys)***，前者是高优先级任务所需的资源被低优先级任务占有，导致效率低下，后者是多个任务反复抢同一把锁

- 同步 synchronization

sleep & wakeup 只是一种同步机制。但任何同步机制都要解决"lost wakeup"问题。Unix直接屏蔽中断，xv6给sleep加了一把锁，而Linux采用***wait queue***机制

- wakeup

如果众多的进程都在等待同一个condition variable，很可能会出现***惊群现象(thundering herd)***，为了避免这一情况，多数的Condition variable会提供两种唤醒方式：signal-唤醒单个，broadcast-唤醒全部

- 终止进程

进程终止是件很复杂的事，多数操作系统都都提供了跳出栈的用户级跳转方法，比如`longjmp`，也提供了完善的信号(signal)机制，这在CSAPP中介绍得很详细，不过xv6并不提供这类机制

- 进程表

xv6在分配进程时需要线性时间扫描进程表，但真实的系统中会有显式的进程空闲列表，可以让系统在常数时间内找到
