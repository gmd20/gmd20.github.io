 
```text
Understanding Linux kernel一书大概说了bio 是怎么传给设备底层驱动的。

14.3.3. Activating the Block Device Driver

http://140.120.7.20/LinuxRef/LinuxKernel/understandlk-chp-14.html#understandlk-chp-14-sect-3.3 



这里会解释block层是怎么把request发给底层的设备驱动的。不过新的3.9的内核这个已经跟书上说的变了好多，

比如block的plug接口应该变的不重要了。参考下面这个文章。

----------------------------

Explicit block device plugging

http://lwn.net/Articles/438256/



plug的用于延时处理，以便收集多个request之后再批量发送给底层磁盘。

不过现在磁盘越来越快，这个plug引入延时缓存的技术已经是没必要的啦？ 至少ssd磁盘应该不会再用了。

所以很多plug延时相关的接口都变了，struct request_queue 里面plug_fn都没有了，相关的定时器也被删了。

如果少数情况要使用plug的话，应该是类似下面这种on-stack plug的用法了。



struct blk_plug plug;

blk_start_plug(&plug);

submit_batch_of_io();

blk_finish_plug(&plug);

这种使用栈上的临时变量plug继续被支持，应该很少用了？ 它把栈上 plug变量指针保存到 current 的tast_struct的plug里面去。后面

blk_flush_plug_list

    queue_unplugged   

访问这个plug，然后也会调用哪个 __blk_run_queue





底层的驱动如果需要延时调用 __blk_run_queue ，比如初始化时设备还没准备好？ 可以使用blk_delay_queue 通知block隔一会再调用blk_run_queue把request发送过来。

---------------------------

scsi层的创建的request_queue 注册的request处理函数是scsi_request_fn。 

block层的__blk_run_queue 函数调用scsi层的scsi_request_fn 函数把request分发给scsi层。





 297/**

 298 * __blk_run_queue_uncond - run a queue whether or not it has been stopped

 299 * @q:  The queue to run

 300 *

 301 * Description:

 302 *    Invoke request handling on a queue if there are any pending requests.

 303 *    May be used to restart request handling after a request has completed.

 304 *    This variant runs the queue whether or not the queue has been

 305 *    stopped. Must be called with the queue lock held and interrupts

 306 *    disabled. See also @blk_run_queue.

 307 */

 308inline void __blk_run_queue_uncond(struct request_queue *q)

 309{

 310        if (unlikely(blk_queue_dead(q)))

 311                return;

 312

 313        /*

 314         * Some request_fn implementations, e.g. scsi_request_fn(), unlock

 315         * the queue lock internally. As a result multiple threads may be

 316         * running such a request function concurrently. Keep track of the

 317         * number of active request_fn invocations such that blk_drain_queue()

 318         * can wait until all these request_fn calls have finished.

 319         */

 320        q->request_fn_active++;

 321        q->request_fn(q);                   ///调用scsi_request_fn 函数

 322        q->request_fn_active--;

 323}

 324

 325/**

 326 * __blk_run_queue - run a single device queue

 327 * @q:  The queue to run

 328 *

 329 * Description:

 330 *    See @blk_run_queue. This variant must be called with the queue lock

 331 *    held and interrupts disabled.

 332 */

 333void __blk_run_queue(struct request_queue *q)

 334{

 335        if (unlikely(blk_queue_stopped(q)))

 336                return;

 337

 338        __blk_run_queue_uncond(q);

 339}

 340EXPORT_SYMBOL(__blk_run_queue);

 341







bio下发的主要调用路径应该和书上讲的还是没多大变化的，还是这样

submit_bio ->

    generic_make_request ->  // 这个函数注释就写的把 bio buffer发送给底层驱动进行IO操作。

       q->make_request_fn    //调用 request_queue的make_request_fn 函数把bio request 插入队列



应该很多request合并，触发request下发到底层驱动的工作都是这个make_request_fn 函数来做的。





scsi 中间层创建request_queue时把这个make_request_fn 函数初始化为blk_queue_bio 函数。

scsi_alloc_queue-> blk_init_allocated_queue ->  blk_queue_make_request -> blk_queue_bio ->__blk_run_queue



blk_queue_bio 里面会尝试合并request的，不过合并不成功，就调用__blk_run_queue把request发给底层驱动。

       __blk_run_queue

如果合并成功了，应该就直接返回了，request在队列里面等待底层驱动自己来取。



不过scsi自己也会从request_queue里面取出request来执行。看源码有两种调用关系是自己主动的blk_run_queue的。

scsi_next_command -> scsi_run_queue -> blk_run_queue -> scsi_request_fn()

scsi_run_host_queues -> scsi_run_queue 



scsi_requeue_command scsi_device_resume等函数也都会调用到scsi_run_queue去取request来运行。



scsi_end_request里面会调用scsi_next_command，然后到blk_run_queue通知block层把下一个命令发送过来。

所以说底层scsi驱动每次自己执行完一个磁盘命令，自己就回去队列里面取下一个来执行的。直到队列为空。

scsi_run_host_queues 这些函数，看样子，会在磁盘启动之类的情况下被执行的吧。

处理request的策略还是靠底层驱动自己来控制了。block层只是第一个request的时候会触发下发动作？ request合并那些动作，会在驱动从队列里面取request的时候，由io scheduler 的各种电梯算法执行。



其他的也有很多触发blk_run_queue的地方，知道的

blk_queue_bio

blk_run_queue_async

flush_end_io

elv_add_request ->__elv_add_request

elevator_switch ->  blk_queue_bypass_start -> __blk_drain_queue

cfq_rq_enqueued -> __blk_run_queue

cfq_schedule_dispatch->cfq_kick_queue







这篇文章好像也不错。

Linux IO 调度层分析 

http://www.360doc.com/content/12/0201/22/2459_183505470.shtml
```
