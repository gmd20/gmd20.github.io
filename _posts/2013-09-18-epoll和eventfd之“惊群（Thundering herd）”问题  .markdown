```text
微博上面有人回复我说epoll的的惊群问题，不是很理解。带着疑问看一下linux kernel的epoll 和 eventfd相关的代码。因为我局的epoll和eventfd一起使用应该是比较简单的。

?

http://lxr.linux.no/linux+v3.11/fs/eventpoll.c

http://lxr.linux.no/linux+v3.11/fs/eventfd.c



------------------

情况1 调用 epoll_wait

epoll_wait

   ep_poll

      __add_wait_queue_exclusive(&ep->wq, &wait);    // 插入队列时有WQ_FLAG_EXCLUSIVE标志，唤醒是只唤醒一个不会有“惊群（Thundering herd）”问题

      freezable_schedule_hrtimeout_range(to, slack,  // 休眠，等待ep_poll_callback函数里面唤醒这个

      __remove_wait_queue(&ep->wq, &wait);   //从休眠中返回就把自己从wait queue队列里面删除了

      ep_send_events            //提交事件到用户空间

          ep_scan_ready_list

       	    ep_read_events_proc

                ep_item_poll

                    file->f_op->poll

                        eventfd_poll

                           poll_wait(file, &ctx->wqh, wait)  //调用_qproc 回调

                              ep_ptable_queue_proc   

      



----------------------------

情况2 调用普通的 poll

ep_eventpoll_poll

    poll_wait   // p->_qproc

    ep_poll_readyevents_proc

      ep_scan_ready_list

          下面同情况1





         



--------------------------

ep_poll_callback   // 由eventfd 里面满足条件时调用这个

   list_add_tail(&epi->rdllink, &ep->rdllist);  //记录触发的事件

   wake_up_locked(&ep->wq); 

         __wake_up_locked((x), TASK_NORMAL, 1)    //1表示只唤醒wait queue里面一个带有WQ_FLAG_EXCLUSIVEwaitqueue

               __wake_up_common(q, mode, nr, 0, NULL);

                //唤醒在epoll_wait ep_poll 里面的等待线程,因为前面是调用了__add_wait_queue_exclusive他、添加进去的



------------

epoll_ctl  函数初始化时把ep_poll_callback回调函数注册到要监控的file的wait queue的里面去。



epoll_ctl

  ep_remove      

     ep_unregister_pollwait 

         ep_remove_wait_queue(pwq); //删掉 ep_insert时注册到file的wait queue

  ep_insert

    ep_set_ffd(&epi->ffd, tfile, fd);

    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc); //注册_qproc 回调

    ep_item_poll(epi, &epq.pt);

          file->f_op->poll

              eventfd_poll

                    poll_wait(file, &ctx->wqh, wait)  //调用_qproc 回调

                        ep_ptable_queue_proc   

                             add_wait_queue(whead, &pwq->wait)



/*

 * This is the callback that is used to add our wait queue to the

 * target file wakeup lists.

 */

ep_ptable_queue_proc

        init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);  //把ep_poll_callback注册为 wait_queue_t q->func 函数，同时初始化 q->flags = 0; 是没有WQ_FLAG_EXCLUSIVE标志的

        add_wait_queue(whead, &pwq->wait);    ///这里的不像上面的__add_wait_queue_exclusive一样设置WQ_FLAG_EXCLUSIVE标志的。

-------------------

eventfd里面满足条件时触发epoll的poll回调函数，唤醒epoll wait



eventfd_write

   wake_up_locked_poll(&ctx->wqh, POLLIN);

       __wake_up_locked_key((x), TASK_NORMAL, (void *) (m))

           __wake_up_common(q, mode, 1, 0, key);  //这里唤醒的时候应该有“惊群”，

                curr->func

                    ep_poll_callback



----------------------





所以根据上面代码知道

1. 如果多个进程或者线程里面 epoll_create创建的同一个“epoll file descriptor”做epoll_wait，那么是没有“惊群（Thundering herd）”问题的。

2. 如果在多个进行或者线程里面自己用 epoll_create创建的各自的“epoll file descriptor”，然后把同一个 eventfd的file descriptor通过epoll_ctl添加到各自的“epoll file descriptor”， 这个多个 “epoll file descriptor”同一个eventfd。

那么但eventfd有事件到来时， 这多个通过epoll wait休眠的进程或者线程是会被同时唤醒的。这就是所谓的“惊群（Thundering herd）”问题。



以后有时间测试一下看看是不是这样。 然后可以看一下socket的file desciptor 是不是行为也是像 eventfd一样的。
```
