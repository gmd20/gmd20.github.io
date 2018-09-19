    




  下载LOFTER我的照片书  |
1. 线程 和执行流程

（1）用户调用io_submit 之后，应该是马上通过异步文件操作，request直接下到文件系统下面去了，文件系统完成了之后，调用aio_complete 通知 aio模块，aio模块会把完成事件放到aio_ring的环形缓存里面，或者用eventfd通知用户程序，或者等待用户自己调用

io_getevent来检测状态。



 （2）然后全系统只有唯一一个workqueue 来执行那些需要重试的request 。 看注释说这样是避免多个request同时执行。 request需要重试的时候的，都是放到这个工作队列等待延时重试的。



2. 锁

   总控结构kioctx 里面有一个spinlock，所有的队列啊，属性的，事件通知啊等都通过这个锁来进行。

   但看上去一个用户进程是可以有多个kioctx的。



3. request和队列。

   异步请求 reqeust就是一个kiocb 结构。 执行队列（io_submit的请求都先放到这个队列里面，等待处理），取消用的队列（基本上request一直都会在这里便于查找取消操作），完成事件的队列。

   这个好像都这样设计的了，我自己写的代码也有这么两个队列，取消队列不错，考虑加到我现在的代码里面去，可以参考一下“撤销机制”。

   完成事件队列的ring buffer也有点意思，



4. 通知或者回调函数。

   因为内核和用户空间的差别别，request执行完了，不是直接调用回调通知用户程序。 不过同步模式时，有唤醒用户进程，evenfd信号通知应该也差不多啦。我现在的代码还是用的c++ 里面的boost::function 来做回调，这样就不需要调用方去检测状态了。



5. 内存管理

   request队列和控制结构的内存都从 独立的aio 名字的kmem_cache 里面申请和释放的。类似应用程序的 “内存池”的概念，我自己的代码也搞了个request的 free list来管理。不知道和直接new的比较差异怎么样，

   request都采用内核常见的引用计数的+ get/put操作的办法来管理。这个好像还不错，特别有多放并行处理的时候，看来我也要尽量用share_ptr才行了。



6. 其他

   最大request个数限制，这个也得引入我的代码里面才行。还是有点用处的。

   相比于用户级的libeio的话，这个确实会像网上高手说的一样，这个的request可以同时大量发下去文件系统层，在那里等待处理。而libeio的request是在用户层的队列里面，由几个工作线程处理完了一个才下发一个给底层文件系统。

另外libeio多了线程抢占锁的情景。这个linux 内核aio的话，只会用完成event和下发event的时候抢占一下。

   另外控制结构和request结构的管理，链表指针等的用法，requesr状态的管理，进程的等待和唤醒，工作队列的使用，retry和cancel方法的设计等还是可以学习参考一下的。



==================================================================================



http://lxr.linux.no/linux+v3.2.9/fs/aio.c

http://lxr.linux.no/linux+v3.2.9/include/linux/aio.h



  /*------ sysctl variables----*/

  static DEFINE_SPINLOCK(aio_nr_lock);

  unsigned long aio_nr;           /* current system wide number of aio requests */

  unsigned long aio_max_nr = 0x10000; /* system wide maximum number of aio requests */ 最大数量限制



  static struct kmem_cache        *kiocb_cachep;   //slab专门为了频繁申请和释放内存提供的设施，类似内存池，request都从这里申请，回收到这里来

  static struct kmem_cache        *kioctx_cachep;





static struct workqueue_struct *aio_wq;     ///全局 work queue





struct kioctx *ctx)  



static struct kioctx *ioctx_alloc(unsigned nr_events)



 179struct kioctx {                              ////kio context  结构，aio的执行环境，总的控制结构//////

 180        atomic_t                users;

 181        int                     dead;

 182        struct mm_struct        *mm;

 183

 184        /* This needs improving */

 185        unsigned long           user_id;

 186        struct hlist_node       list;

 187

 188        wait_queue_head_t       wait;              ///用户的io_getevents请求可能一直等在这里到超时

 189

 190        spinlock_t              ctx_lock;        ///锁是必须的，所有这个kioctx 相关的操作的保护，都是通过这个锁来进行的，队列操作啊，修改request属性的操作啊，等。  lookup_ioctx查找kioctx 结构的时候用的rcu_read_lock

 191

 192        int                     reqs_active;

 193        struct list_head        active_reqs;    /* used for cancellation */  所有的 request （包括开始执行的，已经从run_list里面出来被传给底层文件系统执行那些）应该都在这个链表里面的，需要取消的时候可以通过这里找到对应的request来取消



就可以了。

 194        struct list_head        run_list;       /* used for kicked reqs */        执行的时候，通过这里把在队列上的request取出来，然后调用 aio_read  aio_write  等

 195

 196        /* sys_io_setup currently limits this to an unsigned int */

 197        unsigned                max_reqs;

 198

 199        struct aio_ring_info    ring_info;             ///完成了的aio 事件，放到这里，一个环形缓冲区结构struct aio_ring，通过aio_ring_event 得到下个event的位置，aio_complete 里面会设置，然后用户调用io_getevents来读这里得到完成事件。

 200

 201        struct delayed_work     wq;

 202

 203        struct rcu_head         rcu_head;      //释放kioctx 时的rcu锁，没看到怎么初始化的

 204};



static struct kioctx *ioctx_alloc(unsigned nr_events)    初始化，分配一个kioctx  结构

              初始化 队列等  

             if (aio_setup_ring(ctx) < 0)  初始完成事件的 环形队列，aio_read等完成工作了，会这这队列里面添加完成事件的

             INIT_DELAYED_WORK(&ctx->wq, aio_kick_handler);    初始化，重试request的 work queue的执行函数为aio_kick_handler。  有request 需要重试时，放到这个work queue里面重试



------------------------------------------------------------------

每个aio request就是一个kiocb 来管理的





  53/* is there a better place to document function pointer methods? */

  54/**

  55 * ki_retry     -       iocb forward progress callback

  56 * @kiocb:      The kiocb struct to advance by performing an operation.

  57 *

  58 * This callback is called when the AIO core wants a given AIO operation

  59 * to make forward progress.  The kiocb argument describes the operation

  60 * that is to be performed.  As the operation proceeds, perhaps partially,

  61 * ki_retry is expected to update the kiocb with progress made.  Typically

  62 * ki_retry is set in the AIO core and it itself calls file_operations

  63 * helpers.

  64 *

  65 * ki_retry's return value determines when the AIO operation is completed

  66 * and an event is generated in the AIO event ring.  Except the special

  67 * return values described below, the value that is returned from ki_retry

  68 * is transferred directly into the completion ring as the operation's

  69 * resulting status.  Once this has happened ki_retry *MUST NOT* reference

  70 * the kiocb pointer again.

  71 *

  72 * If ki_retry returns -EIOCBQUEUED it has made a promise that aio_complete()

  73 * will be called on the kiocb pointer in the future.  The AIO core will

  74 * not ask the method again -- ki_retry must ensure forward progress.

  75 * aio_complete() must be called once and only once in the future, multiple

  76 * calls may result in undefined behaviour.

  77 *

  78 * If ki_retry returns -EIOCBRETRY it has made a promise that kick_iocb()

  79 * will be called on the kiocb pointer in the future.  This may happen

  80 * through generic helpers that associate kiocb->ki_wait with a wait

  81 * queue head that ki_retry uses via current->io_wait.  It can also happen

  82 * with custom tracking and manual calls to kick_iocb(), though that is

  83 * discouraged.  In either case, kick_iocb() must be called once and only

  84 * once.  ki_retry must ensure forward progress, the AIO core will wait

  85 * indefinitely for kick_iocb() to be called.

  86 */

  87struct kiocb {

  88        struct list_head        ki_run_list;    //链表头用来把这个request加到 kioctx 的run_list 队列里面去。

  89        unsigned long           ki_flags;

  90        int                     ki_users;       //引用计数 

  91        unsigned                ki_key;         /* id of this request */

  92

  93        struct file             *ki_filp;

  94        struct kioctx           *ki_ctx;        /* may be NULL for sync ops */      这里有kioctx的指针，request传下去的时候，处理request的时候，可以根据这个来找到对应的kioctx

  95        int                     (*ki_cancel)(struct kiocb *, struct io_event *);

  96        ssize_t                 (*ki_retry)(struct kiocb *);

  97        void                    (*ki_dtor)(struct kiocb *);

  98

  99        union {

 100                void __user             *user;

 101                struct task_struct      *tsk;

 102        } ki_obj;

 103

 104        __u64                   ki_user_data;   /* user's data for completion */

 105        loff_t                  ki_pos;

 106

 107        void                    *private;

 108        /* State that we remember to be able to restart/retry  */

 109        unsigned short          ki_opcode;

 110        size_t                  ki_nbytes;      /* copy of iocb->aio_nbytes */

 111        char                    __user *ki_buf; /* remaining iocb->aio_buf */

 112        size_t                  ki_left;        /* remaining bytes */

 113        struct iovec            ki_inline_vec;  /* inline vector */

 114        struct iovec            *ki_iovec;

 115        unsigned long           ki_nr_segs;

 116        unsigned long           ki_cur_seg;

 117

 118        struct list_head        ki_list;        /* the aio core uses this       //用于把request放到kioctx 结构的active_reqs链表里面去，用户调用io_cancel取消时，就可以直接用 lookup_kiocb遍历链表找到kiocb 来cancel了。 aio_complete的时



候才把这个从kioctx的 active_reqs 队列里面拿掉

 119                                                 * for cancellation */

 120        struct list_head        ki_batch;       /* batch allocation */           如果io_submit 批量提交很多个请求的话，会一起申请很多request放到这里，说是为了避免重复多次获取锁和释放锁

 121

 122        /*

 123         * If the aio_resfd field of the userspace iocb is not zero,

 124         * this is the underlying eventfd context to deliver events to.

 125         */

 126        struct eventfd_ctx      *ki_eventfd;                  ///用户设置的 evenfd通知

 127};

 128



















---------------------------------------------------------------------------

static void aio_cancel_all(struct kioctx *ctx)









 883/*

 884 * aio_run_all_iocbs:

 885 *      Process all pending retries queued on the ioctx

 886 *      run list, and keep running them until the list

 887 *      stays empty.

 888 * Assumes it is operating within the aio issuer's mm context.

 889 */

 890static inline void aio_run_all_iocbs(struct kioctx *ctx)         /////////异步执行request



--------------------------------------------------------------

 898/*

 899 * aio_kick_handler:

 900 *      Work queue handler triggered to process pending             ////request 需要超时重试的 在这里运行

 901 *      retries on an ioctx. Takes on the aio issuer's

 902 *      mm context before running the iocbs, so that

 903 *      copy_xxx_user operates on the issuer's address

 904 *      space.

 905 * Run on aiod's context.

 906 */

 907static void aio_kick_handler(struct work_struct *work)







------------------------------------------------

    



SYSCALL_DEFINE2(io_setup, unsigned, nr_events, aio_context_t __user *, ctxp)           //用户每次请求 aio 执行环境的吧

SYSCALL_DEFINE1(io_destroy, aio_context_t, ctx)





--------------------------------------------------------



1757/* sys_io_submit:

1758 *      Queue the nr iocbs pointed to by iocbpp for processing.  Returns

1759 *      the number of iocbs queued.  May return -EINVAL if the aio_context

1760 *      specified by ctx_id is invalid, if nr is < 0, if the iocb at

1761 *      *iocbpp[0] is not properly initialized, if the operation specified

1762 *      is invalid for the file descriptor in the iocb.  May fail with

1763 *      -EFAULT if any of the data structures point to invalid data.  May

1764 *      fail with -EBADF if the file descriptor specified in the first

1765 *      iocb is invalid.  May fail with -EAGAIN if insufficient resources

1766 *      are available to queue any iocbs.  Will return 0 if nr is 0.  Will

1767 *      fail with -ENOSYS if not implemented.

1768 */

1769SYSCALL_DEFINE3(io_submit, aio_context_t, ctx_id, long, nr,                       //用户通过这个提交  aio 请求过来

1770                struct iocb __user * __user *, iocbpp)

1771{

1772        return do_io_submit(ctx_id, nr, iocbpp, 0);

1773}







1794/* sys_io_cancel:

1795 *      Attempts to cancel an iocb previously passed to io_submit.  If

1796 *      the operation is successfully cancelled, the resulting event is

1797 *      copied into the memory pointed to by result without being placed

1798 *      into the completion queue and 0 is returned.  May fail with

1799 *      -EFAULT if any of the data structures pointed to are invalid.

1800 *      May fail with -EINVAL if aio_context specified by ctx_id is

1801 *      invalid.  May fail with -EAGAIN if the iocb specified was not

1802 *      cancelled.  Will fail with -ENOSYS if not implemented.

1803 */

1804SYSCALL_DEFINE3(io_cancel, aio_context_t, ctx_id, struct iocb __user *, iocb,

1805                struct io_event __user *, result)















1854/* io_getevents:

1855 *      Attempts to read at least min_nr events and up to nr events from

1856 *      the completion queue for the aio_context specified by ctx_id. If

1857 *      it succeeds, the number of read events is returned. May fail with

1858 *      -EINVAL if ctx_id is invalid, if min_nr is out of range, if nr is

1859 *      out of range, if timeout is out of range.  May fail with -EFAULT

1860 *      if any of the memory specified is invalid.  May return 0 or

1861 *      < min_nr if the timeout specified by timeout has elapsed

1862 *      before sufficient events are available, where timeout == NULL

1863 *      specifies an infinite timeout. Note that the timeout pointed to by

1864 *      timeout is relative and will be updated if not NULL and the

1865 *      operation blocks. Will fail with -ENOSYS if not implemented.

1866 */

1867SYSCALL_DEFINE5(io_getevents, aio_context_t, ctx_id,

1868                long, min_nr,

1869                long, nr,

1870                struct io_event __user *, events,

1871                struct timespec __user *, timeout)



                            调用  read_events（）-》aio_read_evt() -->aio_ring_event () 查看aio request是不是完成了，把完成事件复制给用户空间

                           read_events  是会根据timeout 一直等到有事件的，会设置wait queue 和io_schedule schedule调度当前出去

                           完成request 会被 aio_complete 放到  struct aio_ring 的队列里面，aio_ring_event函数从里面去取出来



-----------------------------

1700long do_io_submit(aio_context_t ctx_id, long nr,

1701                  struct iocb __user *__user *iocbpp, bool compat)







    struct kioctx *ctx;

    ctx = lookup_ioctx(ctx_id);

    if (unlikely(__get_user(user_iocb, iocbpp + i))) {

    if (unlikely(copy_from_user(&tmp, user_iocb, sizeof(tmp)))) {

    

    ret = io_submit_one(ctx, user_iocb, &tmp, &batch, compat);











----------------------------------



1597static int io_submit_one(struct kioctx *ctx, struct iocb __user *user_iocb,

1598                         struct iocb *iocb, struct kiocb_batch *batch,

1599                         bool compat)



1601        struct kiocb *req;

1602        struct file *file;



     file = fget(iocb->aio_fildes);

     req = aio_get_req(ctx, batch);  /* returns with 2 references to req */

     aio_run_iocb(req);

       spin_lock_irq(&ctx->ctx_lock);

             req->ki_eventfd = eventfd_ctx_fdget((int) iocb->aio_resfd);   ///设置 eventfd，如果用户submit的时候设置了IOCB_FLAG_RESFD标志，完成时候，根据用户需求用eventfd通知用户

             ret = aio_setup_iocb(req, compat);       //设置具体的工作函数是 aio_read 还是 aio_write 等

             aio_run_iocb(req);                ///执行request在这里，调用 aio_read  等干活，都是异步操作，马上返回的

       spin_unlock_irq(&ctx->ctx_lock);

     aio_put_req(req);       /* drop extra ref to req */







-------------------------------





aio_setup_iocb    检查一些设置

                       if (file->f_op->aio_read)

                                kiocb->ki_retry = aio_rw_vect_retry;      



------------------------------------------

aio_rw_vect_retry （）

           这里面回去调用file->f_op->aio_write; 这些实际操作

 if ((iocb->ki_opcode == IOCB_CMD_PREADV) ||

      rw_op = file->f_op->aio_read;

  else

      rw_op = file->f_op->aio_write;



 ret = rw_op(iocb, &iocb->ki_iovec[iocb->ki_cur_seg],

1415                            iocb->ki_nr_segs - iocb->ki_cur_seg,

1416                            iocb->ki_pos);





-----------------------------

 720/* aio_run_iocb

 721 *      This is the core aio execution routine. It is

 722 *      invoked both for initial i/o submission and

 723 *      subsequent retries via the aio_kick_handler.

 724 *      Expects to be invoked with iocb->ki_ctx->lock

 725 *      already held. The lock is released and reacquired

 726 *      as needed during processing.

 727 *

 728 * Calls the iocb retry method (already setup for the         ////////////这里有说明

 729 * iocb on initial submission) for operation specific

 730 * handling, but takes care of most of common retry

 731 * execution details for a given iocb. The retry method

 732 * needs to be non-blocking as far as possible, to avoid

 733 * holding up other iocbs waiting to be serviced by the

 734 * retry kernel thread.

 735 *

 736 * The trickier parts in this code have to do with

 737 * ensuring that only one retry instance is in progress

 738 * for a given iocb at any time. Providing that guarantee

 739 * simplifies the coding of individual aio operations as

 740 * it avoids various potential races.

 741 */

 742static ssize_t aio_run_iocb(struct kiocb *iocb)   这里面就会去调用 初始化好的  ki_retry 回调函数

 743{

 744        struct kioctx   *ctx = iocb->ki_ctx;

 745        ssize_t (*retry)(struct kiocb *);

 746        ssize_t ret;

 747

 748        if (!(retry = iocb->ki_retry)) { /////////////////////////////////retry 指针在这里设置

 749                printk("aio_run_iocb: iocb->ki_retry = NULL\n");

 750                return 0;

 751        }

 752

 753        /*

 754         * We don't want the next retry iteration for this

 755         * operation to start until this one has returned and

 756         * updated the iocb state. However, wait_queue functions

 757         * can trigger a kick_iocb from interrupt context in the

 758         * meantime, indicating that data is available for the next

 759         * iteration. We want to remember that and enable the

 760         * next retry iteration _after_ we are through with

 761         * this one.

 762         *

 763         * So, in order to be able to register a "kick", but

 764         * prevent it from being queued now, we clear the kick

 765         * flag, but make the kick code *think* that the iocb is

 766         * still on the run list until we are actually done.

 767         * When we are done with this iteration, we check if

 768         * the iocb was kicked in the meantime and if so, queue

 769         * it up afresh.

 770         */

 771

 772        kiocbClearKicked(iocb);

 773

 774        /*

 775         * This is so that aio_complete knows it doesn't need to

 776         * pull the iocb off the run list (We can't just call

 777         * INIT_LIST_HEAD because we don't want a kick_iocb to

 778         * queue this on the run list yet)

 779         */

 780        iocb->ki_run_list.next = iocb->ki_run_list.prev = NULL;

 781        spin_unlock_irq(&ctx->ctx_lock);

 782

 783        /* Quit retrying if the i/o has been cancelled */

 784        if (kiocbIsCancelled(iocb)) {

 785                ret = -EINTR;

 786                aio_complete(iocb, ret, 0);

 787                /* must not access the iocb after this */

 788                goto out;

 789        }

 790

 791        /*

 792         * Now we are all set to call the retry method in async

 793         * context.

 794         */

 795        ret = retry(iocb);   /////////////干活 

 796

 797        if (ret != -EIOCBRETRY && ret != -EIOCBQUEUED) {

 798                /*

 799                 * There's no easy way to restart the syscall since other AIO's

 800                 * may be already running. Just fail this IO with EINTR.

 801                 */

 802                if (unlikely(ret == -ERESTARTSYS || ret == -ERESTARTNOINTR ||

 803                             ret == -ERESTARTNOHAND || ret == -ERESTART_RESTARTBLOCK))

 804                        ret = -EINTR;

 805                aio_complete(iocb, ret, 0);  //////////////////完成回调？？？？？

 806        }

 807out:

 808        spin_lock_irq(&ctx->ctx_lock);   //////////锁

 809

 810        if (-EIOCBRETRY == ret) {      ///////////////////////////需要重试的，下面安排重试的

 811                /*

 812                 * OK, now that we are done with this iteration

 813                 * and know that there is more left to go,

 814                 * this is where we let go so that a subsequent

 815                 * "kick" can start the next iteration

 816                 */

 817

 818                /* will make __queue_kicked_iocb succeed from here on */

 819                INIT_LIST_HEAD(&iocb->ki_run_list);

 820                /* we must queue the next iteration ourselves, if it

 821                 * has already been kicked */

 822                if (kiocbIsKicked(iocb)) {

 823                        __queue_kicked_iocb(iocb);                 //其实就是把iocb （一个异步请求）又重新放到执行队列里面去而已。

 824

 825                        /*

 826                         * __queue_kicked_iocb will always return 1 here, because

 827                         * iocb->ki_run_list is empty at this point so it should

 828                         * be safe to unconditionally queue the context into the

 829                         * work queue.

 830                         */

 831                        aio_queue_work(ctx);         调用 queue_delayed_work(aio_wq, &ctx->wq, timeout);    ///有一个全局的工作队列，static struct workqueue_struct *aio_wq;  系统开始时，初始化aio_wq = alloc_workqueue("aio", 0, 1);  



/* used to limit concurrency */

 832                }

 833        }

 834        return ret;

 835}

 836

























 837/*

 838 * __aio_run_iocbs:

 839 *      Process all pending retries queued on the ioctx

 840 *      run list.

 841 * Assumes it is operating within the aio issuer's mm

 842 * context.

 843 */

 844static int __aio_run_iocbs(struct kioctx *ctx)

 845{

 846        struct kiocb *iocb;

 847        struct list_head run_list;

 848

 849        assert_spin_locked(&ctx->ctx_lock);

 850

 851        list_replace_init(&ctx->run_list, &run_list);      /////////////交换list 来运行，这样可能是因为，下面aio_run_iocb 函数会把需要重试的又加到这里来，放到work queue里面去重试，这样不会把需要重试的和没有执行的搞混吧。

 852        while (!list_empty(&run_list)) {

 853                iocb = list_entry(run_list.next, struct kiocb,

 854                        ki_run_list);

 855                list_del(&iocb->ki_run_list);

 856                /*

 857                 * Hold an extra reference while retrying i/o.

 858                 */

 859                iocb->ki_users++;       /* grab extra reference */

 860                aio_run_iocb(iocb);             ////////////////////这个上面那个执行了

 861                __aio_put_req(ctx, iocb); 

 862        }

 863        if (!list_empty(&ctx->run_list))

 864                return 1;

 865        return 0;

 866}







 883/*

 884 * aio_run_all_iocbs:

 885 *      Process all pending retries queued on the ioctx

 886 *      run list, and keep running them until the list

 887 *      stays empty.

 888 * Assumes it is operating within the aio issuer's mm context.

 889 */

 890static inline void aio_run_all_iocbs(struct kioctx *ctx)

 891{

 892        spin_lock_irq(&ctx->ctx_lock);

 893        while (__aio_run_iocbs(ctx))

 894                ;

 895        spin_unlock_irq(&ctx->ctx_lock);

 896}







=============================================================================================================

request的发送流程  aio_run_all_iocbs  -》 __aio_run_iocbs -》aio_run_iocb-》aio_rw_vect_retry  -》file->f_op->aio_read aio_write

==============================================================================================================















 898/*

 899 * aio_kick_handler:

 900 *      Work queue handler triggered to process pending

 901 *      retries on an ioctx. Takes on the aio issuer's

 902 *      mm context before running the iocbs, so that

 903 *      copy_xxx_user operates on the issuer's address

 904 *      space.

 905 * Run on aiod's context.

 906 */

 907static void aio_kick_handler(struct work_struct *work)

 908{

 909        struct kioctx *ctx = container_of(work, struct kioctx, wq.work);

 910        mm_segment_t oldfs = get_fs();

 911        struct mm_struct *mm;

 912        int requeue;

 913

 914        set_fs(USER_DS);                           ////又看到这个 

 915        use_mm(ctx->mm);               //////设置 虚拟内存映射环境？？？？

 916        spin_lock_irq(&ctx->ctx_lock);

 917        requeue =__aio_run_iocbs(ctx);              ////这里也执行 

 918        mm = ctx->mm;

 919        spin_unlock_irq(&ctx->ctx_lock);

 920        unuse_mm(mm);

 921        set_fs(oldfs);

 922        /*

 923         * we're in a worker thread already, don't use queue_delayed_work,

 924         */

 925        if (requeue)

 926                queue_delayed_work(aio_wq, &ctx->wq, 0);

 927}







-------------------------------------

 972/* aio_complete

 973 *      Called when the io request on the given iocb is complete.

 974 *      Returns true if this is the last user of the request.  The 

 975 *      only other user of the request can be the cancellation code.

 976 */

 977int aio_complete(struct kiocb *iocb, long res, long res2)             



     if (is_sync_kiocb(iocb)) {   

       wake_up_process(iocb->ki_obj.tsk);            ///同步模式的

       return 1;                            



     if (iocb->ki_eventfd != NULL)

          eventfd_signal(iocb->ki_eventfd, 1);    //设置了eventfd 通知的



     if (waitqueue_active(&ctx->wait))              ///其他等待的， io_getevents（） 会等在这里，等待完成事件通知

                 wake_up(&ctx->wait);





===================================================









来看看一个 ext4文件系统底层的异步事件的处理，







 231const struct file_operations ext4_file_operations = {

 232        .llseek         = ext4_llseek,

 233        .read           = do_sync_read,

 234        .write          = do_sync_write,

 235        .aio_read       = generic_file_aio_read,

 236        .aio_write      = ext4_file_write,





-------------------------





1384/**

1385 * generic_file_aio_read - generic filesystem read routine

1386 * @iocb:       kernel I/O control block

1387 * @iov:        io vector request

1388 * @nr_segs:    number of segments in the iovec

1389 * @pos:        current file position

1390 *

1391 * This is the "read()" routine for all filesystems

1392 * that can use the page cache directly.

1393 */

1394ssize_t

1395generic_file_aio_read(struct kiocb *iocb, const struct iovec *iov,

1396                unsigned long nr_segs, loff_t pos)

           struct file *filp = iocb->ki_filp;







 /* coalesce the iovecs and go direct-to-BIO for O_DIRECT */

1410        if (filp->f_flags & O_DIRECT) {





1427                                retval = mapping->a_ops->direct_IO(READ, iocb,           ///////iocb 被传到下面去了，

1428                                                        iov, pos, nr_segs);







--------------------







3024static const struct address_space_operations ext4_ordered_aops = {

3025        .readpage               = ext4_readpage,

3026        .readpages              = ext4_readpages,

3027        .writepage              = ext4_writepage,

3028        .write_begin            = ext4_write_begin,

3029        .write_end              = ext4_ordered_write_end,

3030        .bmap                   = ext4_bmap,

3031        .invalidatepage         = ext4_invalidatepage,

3032        .releasepage            = ext4_releasepage,

3033        .direct_IO              = ext4_direct_IO,          /////////////异步调的就是这个了











2981static ssize_t ext4_direct_IO(int rw, struct kiocb *iocb,

2982                              const struct iovec *iov, loff_t offset,

2983                              unsigned long nr_segs)

2984{

2985        struct file *file = iocb->ki_filp;

2986        struct inode *inode = file->f_mapping->host;

2987        ssize_t ret;

2988

2989        /*

2990         * If we are doing data journalling we don't support O_DIRECT

2991         */

2992        if (ext4_should_journal_data(inode))

2993                return 0;

2994

2995        trace_ext4_direct_IO_enter(inode, offset, iov_length(iov, nr_segs), rw);

2996        if (ext4_test_inode_flag(inode, EXT4_INODE_EXTENTS))

2997                ret = ext4_ext_direct_ IO(rw, iocb, iov, offset, nr_segs);              //iocb一直被传下来，最后就会保存到 底层的request结构里面去

2998        else

2999                ret = ext4_ind_direct_IO(rw, iocb, iov, offset, nr_segs);

3000        trace_ext4_direct_IO_exit(inode, offset,

3001                                iov_length(iov, nr_segs), rw, ret);

3002        return ret;

3003}









  85/*

  86 * check a range of space and convert unwritten extents to written.

  87 *

  88 * Called with inode->i_mutex; we depend on this when we manipulate

  89 * io->flag, since we could otherwise race with ext4_flush_completed_IO()

  90 */

  91int ext4_end_io_nolock(ext4_io_end_t *io)



 111        if (io->iocb)

 112                aio_complete(io->iocb, io->result, 0);            /////////////底层文件系统会调用 aio_complete  通知异步事件的完成





------------------------------------

