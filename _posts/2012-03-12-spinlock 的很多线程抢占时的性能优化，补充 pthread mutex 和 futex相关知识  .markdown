```text
    微博里面看到有人说spinlock线程比较多在以前抢占锁时性能比价差。因为之前搜索过c++了spinlock，大多解决办法就是 那种指数的规避的办法咯，就是如果实在不行，就先让cpu指数的等待时间空转几次，实在还是拿不到锁在休眠。可以去看nginx的 core 代码的spinlock文件的实现。像那些boost的spinlock也差不多大，之前说过了。 这样很多个线程都在抢那个锁的时候，不至于都在那里忙等，比较好一点。

      在网上搜索一下 “spinlock Exponential Backoff”  应该就可以找到相关资料了。

      不过 “spinlock Exponential Backoff”  也还是不够好，呵呵，所以有人又搞了 queue lock。 就是大家不要抢咯，一个个排队来拿锁了。应该是这样吧？  就是少了争抢的。我最讨厌那种饭堂吃饭不排队的人了，大家都在那里抢，谁都吃不到，呵呵spinlock 的很多线程抢占时的性能优化 - widebright - widebright的个人空间

       去网上搜索应该可以找到资料吧。“Anderson Queue Lock” “CLH Queue Lock”  还是什么的。

下面这篇文章里面都有提到上面那些知识的，可以先去看一下下面这个。

Spin Locks and Contention  By Guy Kalinski

http://www.cs.tau.ac.il/~afek/Spin%20Locks%20and%20Contention.ppt


2012-03-27 补充 The Art of Multiprocessor Programming.pdf 一书的第7章 “Spin Locks and Co7ntention” 也说倒了“Exponential Backoff” 和“Queue Locks”。后面 的“Chapter Notes”还提到了其他很多的改进，MCSLock,CLHLock HBOLock HCLHLock（Hierarchical Locks）等等改进spinlock性能的尝试。可以去看一下。

 这书还是写的分厂不错的，推荐一下，后面的免锁设计数据结构和硬件原理的附录也不错。

=================================================================

大家很喜欢那spinlock和  mutex 来比较，我个人也是比较喜欢spinlock一点，可能linux内核代码看多了，总是觉得spinlock性能要好。不过这应该分场合，如果按照下面这个。

 

比如这篇 Pthreads并行编程之spin lock与mutex性能对比分析

http://www.parallellabs.com/2010/01/31/pthreads-programming-spin-lock-vs-mutex-performance-analysis/

--------------------------

Fast Userspace Mutexes   

     用户使用futex的时候，大多的锁操作都是在用户操作里面进行的，不用进入内核，所以性能很好吧。然后有碰到大家都在竞争这个锁的时候，才通过futex系统调用进入内核，内核就会把竞争者做个排队，放到队列里面去。然后唤醒的时候，从队列里面依次唤醒。

      看来 “排队”是解决“拥挤”（锁竞争）的好办法啊。spinlock 的很多线程抢占时的性能优化，补充 pthread mutex 和 futex相关知识 - widebright - widebright的个人空间

     根据futex的实现，如果锁竞争不激烈，大多的操作都是在用户级完成的（也是那些原子汇编指令吧），这个跟spinlock应该没有太多的区别，两者性能不会相差的太大。 两者的区别是futex碰到竞争马上会进入内核去排队，而spinlock会忙等一会在那里重试，然后指数级的等待时间都不到，到了一个固定的timeout之后才主动的让内核有切换的机会。这两种办法有各自的优点吧，很难说哪个更好一些。不过两种不同的办法，初略看来是会导致一点不同的区别，futex那种进入内核线程“排队”的做法应该是可以保证各个线程的公平性的，spinlock这种嘛应该是不能保证线程的公平性的，如果有一个线程一直在那里循环很快的拿锁然后释放可能大多时间都会被他抢去了，其他的线程可能很难抢到锁，导致“饥饿”状态。

 

-------------------------

http://lxr.linux.no/linux/Documentation/robust-futexes.txt

这篇文章主要将使用futex的进程意外退出的时候，怎么快速恢复锁的状态，不能拿锁，然后又挂了，别人也没法用了。 不过开始的对futex的介绍还是写的挺好的。

 

  11A futex is in essence a user-space address, e.g. a 32-bit lock variable

  12field. If userspace notices contention (the lock is already owned and

  13someone else wants to grab it too) then the lock is marked with a value

  14that says "there's a waiter pending", and the sys_futex(FUTEX_WAIT)

  15syscall is used to wait for the other guy to release it. The kernel

  16creates a 'futex queue' internally, so that it can later on match up the

  17waiter with the waker - without them having to know about each other.

  18When the owner thread releases the futex, it notices (via the variable

  19value) that there were waiter(s) pending, and does the

  20sys_futex(FUTEX_WAKE) syscall to wake them up.  Once all waiters have

  21taken and released the lock, the futex is again back to 'uncontended'

  22state, and there's no in-kernel state associated with it. The kernel

  23completely forgets that there ever was a futex at that address. This

  24method makes futexes very lightweight and scalable.

 

---------------------------

SYSCALL_DEFINE6(futex

        return do_futex(uaddr, op, val, tp, uaddr2, val2, val3);

 

------------------------

2628long do_futex(u32 __user *uaddr, int op, u32 val, ktime_t *timeout,

2629                u32 __user *uaddr2, u32 val2, u32 val3)

 

2643        switch (cmd) {

2644        case FUTEX_WAIT:

2645                val3 = FUTEX_BITSET_MATCH_ANY;

2646        case FUTEX_WAIT_BITSET:

2647                ret = futex_wait(uaddr, flags, val, timeout, val3);   ///等待 

2648                break;

2649        case FUTEX_WAKE:

2650                val3 = FUTEX_BITSET_MATCH_ANY;

2651        case FUTEX_WAKE_BITSET:

2652                ret = futex_wake(uaddr, flags, val, val3);      //唤醒 

 

-------------------------------

static int futex_wait(u32 __user *uaddr, unsigned int flags, u32 val,

      ret = futex_wait_setup(uaddr, val, flags, &q, &hb);

1902        /* queue_me and wait for wakeup, timeout, or a signal. */

1903        futex_wait_queue_me(hb, &q, to);         ////一直等在这里等待唤醒了，应该

         if (!unqueue_me(&q))

 

         if (!signal_pending(current))       //被信号意外唤醒的，跳到开头再次重试

        

----------------------------------

 

1752/**

1753 * futex_wait_queue_me() - queue_me() and wait for wakeup, timeout, or signal

1754 * @hb:         the futex hash bucket, must be locked by the caller

1755 * @q:          the futex_q to queue up on

1756 * @timeout:    the prepared hrtimer_sleeper, or null for no timeout

1757 */

1758static void futex_wait_queue_me(struct futex_hash_bucket *hb, struct futex_q *q,

1759                                struct hrtimer_sleeper *timeout)

1760{

1761        /*

1762         * The task state is guaranteed to be set before another task can

1763         * wake it. set_current_state() is implemented using set_mb() and

1764         * queue_me() calls spin_unlock() upon completion, both serializing

1765         * access to the hash list and forcing another memory barrier.

1766         */

1767        set_current_state(TASK_INTERRUPTIBLE);

1768        queue_me(q, hb);

 

-------------------------

1478/**

1479 * queue_me() - Enqueue the futex_q on the futex_hash_bucket

1480 * @q:  The futex_q to enqueue

1481 * @hb: The destination hash bucket

1482 *

1483 * The hb->lock must be held by the caller, and is released here. A call to

1484 * queue_me() is typically paired with exactly one call to unqueue_me().  The

1485 * exceptions involve the PI related operations, which may use unqueue_me_pi()

1486 * or nothing if the unqueue is done as part of the wake process and the unqueue

1487 * state is implicit in the state of woken task (see futex_wait_requeue_pi() for

1488 * an example).

1489 */

1490static inline void queue_me(struct futex_q *q, struct futex_hash_bucket *hb)

1491        __releases(&hb->lock)

1492{

1493        int prio;

1494

1495        /*

1496         * The priority used to register this element is

1497         * - either the real thread-priority for the real-time threads

1498         * (i.e. threads with a priority lower than MAX_RT_PRIO)

1499         * - or MAX_RT_PRIO for non-RT threads.

1500         * Thus, all RT-threads are woken first in priority order, and

1501         * the others are woken last, in FIFO order.

1502         */

1503        prio = min(current->normal_prio, MAX_RT_PRIO);

1504

1505        plist_node_init(&q->list, prio);

1506        plist_add(&q->list, &hb->chain);   ///加到等待队列里面去，plist是一个优先级队列，

1507        q->task = current;

1508        spin_unlock(&hb->lock);

1509}

 

----------------------------------------
```
