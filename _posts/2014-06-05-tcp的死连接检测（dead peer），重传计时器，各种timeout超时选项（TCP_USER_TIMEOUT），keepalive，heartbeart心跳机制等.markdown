```text
    




  下载LOFTER我的照片书  |
1.tcp超时重传设置
--------------------
root@debian02:/opt# cat /proc/sys/net/ipv4/tcp_retries
tcp_retries1  tcp_retries2
root@debian02:/opt# cat /proc/sys/net/ipv4/tcp_retries
tcp_retries1  tcp_retries2
root@debian02:/opt# cat /proc/sys/net/ipv4/tcp_retries2
15

默认的超时重传时间，根据上面那两个参数计算出来大概是15 分钟左右。可以设置的小一些，但好像一般也不能设置的太短了。
如果网络强制关闭，系统没来得及发FIN或者Reset给对端tcp，那么tcp应该是不能检测到这种死连接的。
如果一直往这个死连接上面发送数据，数据发送不出去就会重传，那么应该等到这个重传超时之后才能检查到错误。

重传还是不成功，按照Linux的实现，所有没有发送出去的数据都标识为丢失，对应tcp socket要进入loss state（Congestion
control state machine的一种状态）， 接下来的就是等待恢复了 “TCP Loss Recovery”


2.对应的tcp keep-alive全局设置
----------------------------
root@debian02:/opt# cat /proc/sys/net/ipv4/tcp_keepalive_time
7200        默认两个小时才会开始发keepalive探测包
root@debian02:/opt# cat /proc/sys/net/ipv4/tcp_keepalive_probes
9
root@debian02:/opt# cat /proc/sys/net/ipv4/tcp_keepalive_intvl
75

很多网上例子设置很短的tcp_keepalive_time值，比如几秒，来用于探测tcp死连接。但好像说这个其实不是很好的做法，tcp keep alive最开始设计时候本来就不是打算这么用的。我觉得最好还是上层应用自己实现自己的heartbeat机制，自己双向不停的每隔一段时间就发心跳检查包。只有双向的心跳包都收到了才当作连接正常，这个应该是最可靠的了吧。


为每个tcp socket设置这个keep alive参数的代码
int keep_alive = 1;
int keep_idle = 5, keep_interval = 1, keep_count = 3;
int ret = 0;

if (-1 == (ret = setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &keep_alive,
    sizeof(keep_alive)))) {
    fprintf(stderr, "[%s %d] set socket to keep alive error: %s", __FILE__,
        __LINE__, ERRSTR);
}
if (-1 == (ret = setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE, &keep_idle,
    sizeof(keep_idle)))) {
    fprintf(stderr, "[%s %d] set socket keep alive idle error: %s", __FILE__,
        __LINE__, ERRSTR);
}
if (-1 == (ret = setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &keep_interval,
    sizeof(keep_interval)))) {
    fprintf(stderr, "[%s %d] set socket keep alive interval error: %s", __FILE__,
        __LINE__, ERRSTR);
}
if (-1 == (ret = setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT, &keep_count,
    sizeof(keep_count)))) {
    fprintf(stderr, "[%s %d] set socket keep alive count error: %s", __FILE__,
        __LINE__, ERRSTR);
}

参考
http://blog.leeyiw.org/tcp-keep-alive/
http://stackoverflow.com/questions/5907527/application-control-of-tcp-retransmission-on-linux
这文章里面的说明，如果client由于各种原因（比如直接断点）导致的死连接的时候，server端如果有数据发送出去，但还client还没有回应ack的时候，keep alive机制不会被激活，而是重传机制先起作用，先等待15分钟重传超时。如果是server开始没有unack的数据，那么开始发keep alive ，然后client这时又没有任何回应的话，keep alive是会一直发下去的，可能这个keep alive已经在之前被ack过了（重发最后发送给对方一个字节，要求对方重新回复ack），所以这个keep alive没有收到client的ack的话，也是不会触发重传的。 实际测试看到如果client没有任何响应，这个keep alive按照间隔时间重试指定次数之后，应该就会关闭了。这个过程中没有触发重发机制。
所以keep alive要想启动作用能探测到死连接的话，关键是本方没有unack的数据，不然不可避免的等待这个重传超时。


2014-08-04补充：

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




3. 新的TCP_USER_TIMEOUT 选项
---------------------------
int tcp_timeout        =10000; //10 seconds before aborting a write()
result = setsockopt(socket_fd, SOL_TCP, TCP_USER_TIMEOUT, &tcp_timeout, sizeof(int));

参考
TCP User Timeout Option
http://tools.ietf.org/html/rfc5482

tcp的设置选项的说明
http://man7.org/linux/man-pages/man7/tcp.7.html

commit dca43c75e7e545694a9dd6288553f55c53e2a3a3 Author: Jerry Chu Date: Fri Aug 27 19:13:28 2010 +0000

tcp: Add TCP_USER_TIMEOUT socket option.

This patch provides a "user timeout" support as described in RFC793. The
socket option is also needed for the the local half of RFC5482 "TCP User
Timeout Option".

TCP_USER_TIMEOUT is a TCP level socket option that takes an unsigned int,
when > 0, to specify the maximum amount of time in ms that transmitted
data may remain unacknowledged before TCP will forcefully close the
corresponding connection and return ETIMEDOUT to the application. If
0 is given, TCP will continue to use the system default.

Increasing the user timeouts allows a TCP connection to survive extended
periods without end-to-end connectivity. Decreasing the user timeouts
allows applications to "fail fast" if so desired. Otherwise it may take
upto 20 minutes with the current system defaults in a normal WAN
environment.

The socket option can be made during any state of a TCP connection, but
is only effective during the synchronized states of a connection
(ESTABLISHED, FIN-WAIT-1, FIN-WAIT-2, CLOSE-WAIT, CLOSING, or LAST-ACK).
Moreover, when used with the TCP keepalive (SO_KEEPALIVE) option,
TCP_USER_TIMEOUT will overtake keepalive to determine when to close a
connection due to keepalive failure.

The option does not change in anyway when TCP retransmits a packet, nor
when a keepalive probe will be sent.

This option, like many others, will be inherited by an acceptor from its
listener.

Signed-off-by: H.K. Jerry Chu <hkchu@google.com>
Signed-off-by: David S. Miller <davem@davemloft.net>

这个的意思是说如果有数据已经发送出去了，对方还没有回应ack的话，就会在这个的设置时间到了之后就close这个socket。
按照我的理解：
（1）.那么这个起作用前提是你要有数据发送一直在往外发才行。如果正在“重传”，那肯定有数据还没有收到ack，所以设置这个应该是可以规避那个15分钟的重传时间的。 前面的注释说了，这个选项是不影响tcp什么时候开始重传数据的。

（2）.keep alive也是会主动发的数据的（（重发最后发送给对方一个字节，要求对方重新回复ack）），如果你一直没有往外发送数据。也可以等这个keep alive 的包往外发的时候，这个keep alive的包就没有ack了。这时就会触发这个设置了。如果这个时间比keep alive的超时时间短，这个应该是先起作用的。 但keep live什么时候才开始探测包，不受这个参数控制，而至由keep alive对应的参数来控制的。、  但就不知道所有数据都已经被ack回应过了，这个时候如果没有keep alive，这个选项也不起作用了吧，探测不出已经死的连接。除非上层再次往下面发送数据，这个就是“心跳”包要每隔一段时间就要重发一次的作用吧。

（3）.  这个选项可以在数据没有收到对方ack的时候起作用，那就可以避免前面说的存在未ack的消息keepalive 机制不起作用取药先重传的问题。keep alive要和这个选项配合使用才能绕过那个15分钟的重传超时。


4.检测tcp发送队列的长度
----------------------
 ioctl() SIOCOUTQ的选项可以检测socket的发送队列长度，如果长时间都没变化。说明没有收到对方发过来的ack了，
可以自己把tcp socket给关闭。

不过这个看起来不是很严谨的做法。




5. 每个sockete的send write操作超时的选项
----------------------------
setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv))
setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(tv))

设置了这个应该就可以指定了每个send 和write操作最大的允许的阻塞时间，
但对于send来说，如果tcp缓存可以装的下这次的数据，还是send不会阻塞的，除非一直send，然后把tcp/socket buffer给塞满了，这个时候的send才能返回错误。 write提前快速返回超时错误，可能也不是tcp连接断开了，可能真是没有数据过来也是可能的。




6. tcp connect操作的超时控制
----------------------------

上面的重传超时和 TCP_USER_TIMEOUT 看上去都不是控制 建立tcp socket连接时的超时的。
linux上面对应的重试发送syn包的应该是这个，默认时间的是20秒左右。

root@debian02:/opt# cat /proc/sys/net/ipv4/tcp_syn_retries
6
root@debian02:/opt# cat /proc/sys/net/ipv4/tcp_synack_retries
5


7 模拟这个tcp的连接丢失问题
---------------------------
让计算机休眠一会，或者虚拟机里面，比如virtualbox就有一个 “暂停”
虚拟机的功能。这样就会出现tcp死链接。网上有人这么说，我实际测试也
发现休眠确实可以的。 直接拔了网线不知道行不行，也有拔远端的也可以？
但有些时候有可能触发路由不可达的icmp错误？

8 TCP and heartbeats
---------------------
http://250bpm.com/blog:22
这篇文章提到SCTP协议的heartbeat机制也许参考一下，如果需要在tcp上面应用里实现一个heartbeat机制的话。


有用的参考
==========

TCP Keepalive HOWTO
http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/index.html

linux的一些tcp选项的参数说明
http://man7.org/linux/man-pages/man7/tcp.7.html

TCP的RFC
TRANSMISSION CONTROL PROTOCOL
http://tools.ietf.org/html/rfc793

经典的制作好像讨论了很多超时方面的问题
TCP/IP Illustrated, Volume 1 The Protocols
W. Richard Stevens
http://www.pcvr.nl/tcpip/

Linux TCP
http://people.cs.clemson.edu/~westall/853/linuxtcp.pdf

对应的内核代码
http://lxr.linux.no/linux+v3.14.5/net/ipv4/tcp_timer.c
tcp_retransmit_timer
tcp_keepalive_timer
等相关函数

```
