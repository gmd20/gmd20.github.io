    




  下载LOFTER我的照片书  |
做一个网络断开导致tcp连接错误的测试，
程序本来是想依赖keepalive来探测到tcp死链接，结果测试中发现的情况有3种：
1.
如果网络断开时，tcp连接双方没有数据通讯，那么等tcp keepalive时间到了之后，
keepalive timer生效发送几次keepalive探测包之后对方没有回应，内核会关闭连接。
这个keepalive超时的总时间除了keepalive相关的几个选项之后，用户设置的
TCP_USER_TIMEOUT选项的超时时间也起作用。这样可以快速探测到网络错误。
2.
如果网络断开时，tcp有网络包已经发送出去了，但还没有收到对方的ack。
那么内核启动重传定时器（retransmission timer）。这个重传超时时间受
tcp_retries1 tcp_retries2两个选项控制，默认15分钟左右。用户设置的TCP_USER_TIMEOUT
超时也会使的重传定时器在重传失败之后快速返回错误。
3.
如果网络断开时，有数据需要发送，但还没有实际发送出去的（没有第2种情况的
有unack的数据包已经在网络上）。 那么tcp会尝试发送网络包出去的时候得到失败的返回
（通过iptables直接丢掉某个端口的网络包就是这种情况，tcp层 会得到netfilter丢包
导致发送skb错误通知）。
这种情况下内核在发送skb错误之后，会启动probe定时器（坚持定时器TCP Persist
Timer http://www.pcvr.nl/tcpip/tcp_pers.htm）. 也是认为错误的时候是由于0 windows size
导致的？？ 虽然这个keepalive 定时器也会被周期性的触发，但优先起作用是probe定时器。
probe定时器的超时和默认的重传15分钟超时一样受两个retries选项控制。但probe不受用户设置的
TCP_USER_TIMEOUT 超时控制。所以这种iptables导致连接失效的情况，linux的tcp要过
15分钟左右才能探测到网络错误关闭连接。

所以最可能的快速检测tcp网络错误的方式还是 上层应用自己实现自己的类似SCTP协议的
heartbeat机制，自己双向不停的每隔一段时间就发心跳检查包。只有双向的心跳包都收到
了才当作连接正常”。


下面是测试过程中找到的一些跟踪/查看linux的tcp连接状态（主要是为了查看定时器）的工具和方法。

内核proc文件有导出tcp的状态的
============================
cat /proc/net/tcp
http://lxr.free-electrons.com/source/net/ipv4/tcp_ipv4.c?v=3.2
触发函数
static void get_tcp4_sock(struct sock *sk, struct seq_file *f, int i, int
    *len)

这个会打印具体的tcp连接的状态，
root@debian01:/home/bright# cat /proc/net/tcp
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode
   0: 00000000:8B45 00000000:0000 0A 00000000:00000000 00:00000000 00000000   105        0 5822 1 f7275580 100 0 0 10 0
   7: 0100007F:0019 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 6871 1 f5faeac0 100 0 0 10 0
   8: 0100007F:177A 00000000:0000 0A 00000000:00000000 00:00000000 00000000  1000        0 798696 1 f7658ac0 100 0 0 10 0
  17: 6638A8C0:03C0 0138A8C0:3382 01 00000000:00000000 02:000002A4 00000000     0        0 771794 2 f7388580 40 4 30 10 -1
  18: 6638A8C0:0016 0138A8C0:6BAE 01 00000034:00000000 01:0000002A 00000000     0        0 798586 4 f77a0580 42 4 31 10 -1

 tr  是对于的timer类型，比如1是 重传定时器，2是probe0定时器，0
 是没有定时器等。参考get_tcp4_sock 的源码。
 timeout对应icsk_probes_out，表示keepalive超时了几次等等。


netstat也有很很多tcp状态的信息，比如查看当前tcp连接定时器状态的
===========================================
“netstat --timers” 也可以查看各个timer的状态， 应该就是直接读取/proc/net/tcp
文件信息得到的吧。




内核源码的相关函数
==================
内核相关的3种定时器函数
http://lxr.free-electrons.com/source/net/ipv4/tcp_timer.c?v=3.2

void tcp_retransmit_timer(struct sock *sk)
static void tcp_keepalive_timer (unsigned long data)
static void tcp_probe_timer(struct sock *sk)

--------
tcp_sock, inet_connection_sock, inet_sock, sock结构的关系，
这几个结构包含大量的状态信息，因为结构都是放在结构嵌套的最开始的位置的，
所以这个4个struct其实地址都是是一样的，可以简单的通过地址转换得到各个结构指针。
struct tcp_sock {
  struct inet_connection_sock     inet_conn {
    struct inet_sock          icsk_inet {
      struct inet_sock {
        struct sock             sk {
           struct sk_buff          *sk_send_head; //还有tcp网络包需要发送出去，待发送的skb的队列头。
        }
        __be16                  inet_dport;
        __be16                  inet_sport;
      }
      u32                       icsk_user_timeout; // TCP_USER_TIMEOUT用户自定义超时设置
      __u8                      icsk_probes_out;   // keepalive 或者probe的包已经发送了几次
    }
  }
  u32     packets_out;  //时候有网络包已经发送出去
}


指针转换的例子，参考内核源码
517 static void tcp_keepalive_timer (unsigned long data)
518 {
519         struct sock *sk = (struct sock *) data;
520         struct inet_connection_sock *icsk = inet_csk(sk);
521         struct tcp_sock *tp = tcp_sk(sk);





---------------------------------------


http://lxr.free-electrons.com/source/net/ipv4/tcp_timer.c?v=3.2
517 static void tcp_keepalive_timer (unsigned long data)
518 {
519         struct sock *sk = (struct sock *) data;
520         struct inet_connection_sock *icsk = inet_csk(sk);
521         struct tcp_sock *tp = tcp_sk(sk);
522         u32 elapsed;
523
524         /* Only process if socket is not in use. */
525         bh_lock_sock(sk);
526         if (sock_owned_by_user(sk)) {
527                 /* Try again later. */
528                 inet_csk_reset_keepalive_timer (sk, HZ/20);
529                 goto out;
530         }
531
532         if (sk->sk_state == TCP_LISTEN) {
533                 tcp_synack_timer(sk);
534                 goto out;
535         }
536
537         if (sk->sk_state == TCP_FIN_WAIT2 && sock_flag(sk, SOCK_DEAD)) {
538                 if (tp->linger2 >= 0) {
539                         const int tmo = tcp_fin_time(sk) - TCP_TIMEWAIT_LEN;
540
541                         if (tmo > 0) {
542                                 tcp_time_wait(sk, TCP_FIN_WAIT2, tmo);
543                                 goto out;
544                         }
545                 }
546                 tcp_send_active_reset(sk, GFP_ATOMIC);
547                 goto death;
548         }
549
550         if (!sock_flag(sk, SOCK_KEEPOPEN) || sk->sk_state == TCP_CLOSE)
551                 goto out;
552
553         elapsed = keepalive_time_when(tp);
554
555         /* It is alive without keepalive 8) */
556         if (tp->packets_out || tcp_send_head(sk))  // 如果有数据包在路上，或者tcp_send_head这个等待发送的skb队列不为空，就忽略keepalive 机制

557                 goto resched;
558
559         elapsed = keepalive_time_elapsed(tp);
560
561         if (elapsed >= keepalive_time_when(tp)) {
562                 /* If the TCP_USER_TIMEOUT option is enabled, use that
563                  * to determine when to timeout instead.
564                  */
565                 if ((icsk->icsk_user_timeout != 0 &&
566                     elapsed >= icsk->icsk_user_timeout &&
567                     icsk->icsk_probes_out > 0) ||
568                     (icsk->icsk_user_timeout == 0 &&              // 用户设置的TCP_USER_TIMEOUT
569                     icsk->icsk_probes_out >= keepalive_probes(tp))) {
570                         tcp_send_active_reset(sk, GFP_ATOMIC);
571                         tcp_write_err(sk);
572                         goto out;
573                 }
574                 if (tcp_write_wakeup(sk) <= 0) {
575                         icsk->icsk_probes_out++;
576                         elapsed = keepalive_intvl_when(tp);
577                 } else {
578                         /* If keepalive was lost due to local congestion,
579                          * try harder.
580                          */
581                         elapsed = TCP_RESOURCE_PROBE_INTERVAL;
582                 }
583         } else {
584                 /* It is tp->rcv_tstamp + keepalive_time_when(tp) */
585                 elapsed = keepalive_time_when(tp) - elapsed;
586         }
587
588         sk_mem_reclaim(sk);
589
590 resched:
591         inet_csk_reset_keepalive_timer (sk, elapsed);
592         goto out;
593
594 death:
595         tcp_done(sk);
596
597 out:
598         bh_unlock_sock(sk);
599         sock_put(sk);
600 }




271 static void tcp_probe_timer(struct sock *sk)
272 {
273         struct inet_connection_sock *icsk = inet_csk(sk);
274         struct tcp_sock *tp = tcp_sk(sk);
275         int max_probes;
276 
277         if (tp->packets_out || !tcp_send_head(sk)) {   // packets_out 表示有数据包在路上了，   tcp_send_head 是等待发送skb队列，不为空表示有需要发送出去的数据包，才去做probe
278                 icsk->icsk_probes_out = 0;
79                 return;
280         }
281 
282         /* *WARNING* RFC 1122 forbids this
283          *
284          * It doesn't AFAIK, because we kill the retransmit timer -AK
285          *
286          * FIXME: We ought not to do it, Solaris 2.5 actually has fixing
287          * this behaviour in Solaris down as a bug fix. [AC]
288          *
289          * Let me to explain. icsk_probes_out is zeroed by incoming ACKs
290          * even if they advertise zero window. Hence, connection is killed only
291          * if we received no ACKs for normal connection timeout. It is not killed
292          * only because window stays zero for some time, window may be zero
293          * until armageddon and even later. We are in full accordance
294          * with RFCs, only probe timer combines both retransmission timeout
295          * and probe timeout in one bottle.                             --ANK
296          */
297         max_probes = sysctl_tcp_retries2;
298 
299         if (sock_flag(sk, SOCK_DEAD)) {
300                 const int alive = ((icsk->icsk_rto << icsk->icsk_backoff) < TCP_RTO_MAX);
301 
302                 max_probes = tcp_orphan_retries(sk, alive);
303 
304                 if (tcp_out_of_resources(sk, alive || icsk->icsk_probes_out <= max_probes))
305                         return;
306         }
307 
308         if (icsk->icsk_probes_out > max_probes) {
309                 tcp_write_err(sk); //这上面的等于超时重传的时间，不判断TCP_USER_TIMEOUT
310         } else {
311                 /* Only send another probe if we didn't close things up. */
312                 tcp_send_probe0(sk);
313         }
314 }



void tcp_retransmit_timer(struct sock *sk)  和
static void tcp_keepalive_timer (unsigned long data)
检查用户自定义设置的TCP_USER_TIMEOUT超时设置，
但tcp_probe_timer 不忽略TCP_USER_TIMEOUT超时时间的检查，默认的probe超时时间
和正常的重传超时是差不多的，为15分钟左右。

如果probe和keepalive 定时器两个同时触发。keepalive定时器的超时检查会失去作用。
tcp_keepalive_timer 里面的 if (tp->packets_out || tcp_send_head(sk))
检查tcp_send_head 会检查有数据需要发送，直接跳出keepalive的超时机制。

----------
启动probe定时器 路径1
tcp_ack
  tcp_ack_probe    // 如果等待发送skb的序列号大于 tcp的 snd_wnd,  即windows
  size没法容纳指定skb发送出去的时候，就设置probe定时器，发送windows
  size的probe网络包

----------------
启动probe定时器 路径2
__tcp_push_pending_frames
   if (tcp_write_xmit(sk, cur_mss, nonagle, 0, GFP_ATOMIC)) //如果发送失败就启动proble定时器
    tcp_check_probe_timer
      inet_csk_reset_xmit_timer


1819 /* Push out any pending frames which were held back due to
1820  * TCP_CORK or attempt at coalescing tiny packets.
1821  * The socket must be locked by the caller.
1822  */
1823 void __tcp_push_pending_frames(struct sock *sk, unsigned int cur_mss,
1824                                int nonagle)
1825 {
1826         /* If we are closed, the bytes will have to remain here.
1827          * In time closedown will finish, we empty the write queue and
1828          * all will be happy.
1829          */
1830         if (unlikely(sk->sk_state == TCP_CLOSE))
1831                 return;
1832 
1833         if (tcp_write_xmit(sk, cur_mss, nonagle, 0, GFP_ATOMIC))
1834                 tcp_check_probe_timer(sk); //如果上面tcp_write_xmit发送失败就调用probe定时器
1835 }


862 static inline void tcp_check_probe_timer(struct sock *sk)
863 {
864         const struct tcp_sock *tp = tcp_sk(sk);
865         const struct inet_connection_sock *icsk = inet_csk(sk);
866 
867         if (!tp->packets_out && !icsk->icsk_pending) //如果packets_out，那应该是有数据发送出去了，还没有ack的，应该是重传定时器起作用了。
868                 inet_csk_reset_xmit_timer(sk, ICSK_TIME_PROBE0, //启动probe定时器
869                                           icsk->icsk_rto, TCP_RTO_MAX);
870 


 static int tcp_transmit_skb}
       err = icsk->icsk_af_ops->queue_xmit(skb, &inet->cork.fl);
       如果queue_xmit 失败就触发probe

 http://lxr.free-electrons.com/source/net/ipv4/tcp_ipv4.c?v=3.2#L1830
 const struct inet_connection_sock_af_ops ipv4_specific = {
 .queue_xmit        = ip_queue_xmit,


tcp层可以在发送的时候得到netfilter丢包的错误返回，进而启动proble定时器的调用流程。
tcp_write_xmit
  tcp_transmit_skb
    icsk->icsk_af_ops->queue_xmit
      ip_queue_xmit
         ip_local_out
            __ip_local_out
               nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT




systemtap跟踪tcp连接的状态
==========================
systemtap 自带 tcp_trace.stp call_graph.stp， tcp_monitor.stp
等几个例子, 特别是tcp_trace.stp 这个简直是追踪tcp连接状态的神器。

tcp_trace可以指定过滤端口和ip，打印出tcp连接的所有状体变动，可以看到状态、各种
定时器的启动和变化，接受和发送buffer和窗口的大小等。
脚本里面有示范用法和各个参数的详细说明，比如下面这个可以用于实时查看timer的变化：
                                   source addr  remote addr
stap tcp_trace.stp filter=all  *.*.*.*:*-*.*.*.*:10011
stap tcp_trace.stp filter=state,timer,txq  *.*.*.*:*-192.168.56.1:10011
stap tcp_trace.stp filter=state,timer,txq,snd_wnd   *.*.*.*:*-192.168.56.1:10011


可以根据需要在这个脚本基础增加更多信息的输出:

systemtap自带了很多内置函数，比如直接用tcp_sock获取端口的:
laddr = tcpmib_local_addr(sk);
raddr = tcpmib_remote_addr(sk);
lport = tcpmib_local_port(sk);
rport = tcpmib_remote_port(sk);
其他ip地址等函数可以参考systamp的文档。


可以用systemtap的 @cast语法转换函数指针类型，打印不同结果的成员，比如下面这个用法
probe kernel.function("tcp_keepalive_timer")
{
	if ( !timer_flg ) next;

	sk = $data;
	key = filter_key(sk)
	if ( !key ) next;

  printf ("icsk_probes_out=%d, packets_out=%d, sk_send_head=%p\n",
      @cast($data, "inet_connection_sock")->icsk_probes_out,
      @cast($data, "tcp_sock")->packets_out,
      @cast($data, "sock")->sk_send_head
      );

	is_packet_updated(key,sk)
	length[key] = 0
	tx_timer[key] = 4;
	print_packet_info(key, 1)
	tx_timer[key] = 0;
}



可以根据 call_graph.stp  和para-callgraph.stp
脚本为模板，打印对应的tcp函数相互调用过程的trace

-----------------call_graph.stp -----------------------
// stap -v call_graph.stp  ssh

function is_this:long(name:string)
{
	return isinstr(name, @1)
}

probe  kernel.function("*@net/socket.c").call
{
	if (is_this(execname()))
		printf("%s --> %s\n", thread_indent(4), probefunc())
}

probe  kernel.function("*@net/socket.c").return
{
	if (is_this(execname()))
		printf("%s <-- %s\n", thread_indent(-4), probefunc())
}

probe  kernel.function("*@net/ipv4/tcp_input.c").call
{
	if (is_this(execname()))
		printf("%s --> %s\n", thread_indent(4), probefunc())
}

probe  kernel.function("*@net/ipv4/tcp_input.c").return
{
	if (is_this(execname()))
		printf("%s <-- %s\n", thread_indent(-4), probefunc())
}

--------------para-callgraph.stp----------------------------------

#! /usr/bin/env stap

function trace(entry_p, extra) {
  %( $# > 1 %? if (tid() in trace) %)
  printf("%s%s%s %s\n",
         thread_indent (entry_p),
         (entry_p>0?"->":"<-"),
         ppfunc (),
         extra)
}


%( $# > 1 %?
global trace
probe $2.call {
  trace[tid()] = 1
}
probe $2.return {
  delete trace[tid()]
}
%)

probe $1.call   { trace(1, $$parms) }
probe $1.return { trace(-1, $$return) }

// 调用的例子第一个参数是需要记录的函数。第二个参数是触发点。
// stap para-callgraph.stp 'kernel.function("*@fs/*.c")' 'kernel.function("sys_read")':
------------------------------------------------------


结合使用tcp_trace.stp 和 call_graph
两个脚本，我觉的应该时所有的tcp连接状态都可以分析的清楚了吧。
