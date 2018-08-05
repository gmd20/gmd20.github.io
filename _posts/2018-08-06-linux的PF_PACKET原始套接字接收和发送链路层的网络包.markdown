PF_PACKET socket可以用来接收和发送链路层的网络包，让你直接在二层的mac地址头后面
自由构造任意的数据直接投递给网卡驱动，应该是绕过netfilter和QDISC流控等协议层的。
这个可以用来直接发送ARP/ICMP/IGMP/802.1q vlan头的网络包等。

```text
       Packet sockets are used to receive or send raw packets at the device
       driver (OSI Layer 2) level.  They allow the user to implement
       protocol modules in user space on top of the physical layer.
```


# 创建
```c
socket(PF_PACKET, SOCK_PACKET, htons((short)ETH_P_ALL))
```
这里的ETH_P_ALL应该对应netif_receive_skb 里面的前面的那个proto的链表吧，一个是
ALL，另外也可以具体协议的， SOCK_PACKET 类似抓包工具就是用这个吧，和SOCK_RAW很
类似，但文档说确实不一样的，搞不懂。


```c
/**
 * enum sock_type - Socket types
 * @SOCK_STREAM: stream (connection) socket
 * @SOCK_DGRAM: datagram (conn.less) socket
 * @SOCK_RAW: raw socket
 * @SOCK_RDM: reliably-delivered message
 * @SOCK_SEQPACKET: sequential packet socket
 * @SOCK_DCCP: Datagram Congestion Control Protocol socket
 * @SOCK_PACKET: linux specific way of getting packets at the dev level.
 *		  For writing rarp and other similar things on the user level.
 *
 * When adding some new socket type please
 * grep ARCH_HAS_SOCKET_TYPE include/asm-* /socket.h, at least MIPS
 * overrides this enum for binary compat reasons.
 */
enum sock_type {
	SOCK_STREAM	= 1,
	SOCK_DGRAM	= 2,
	SOCK_RAW	= 3,
	SOCK_RDM	= 4,
	SOCK_SEQPACKET	= 5,
	SOCK_DCCP	= 6,
	SOCK_PACKET	= 10,
};
```


# bind绑定
一般是抓链路层的网络包，所以一般的bind都是指定网络接口。不过应该也可以制定其他类型的地址的吧。
另外可以用PACKET_ADD_MEMBERSHIP socket选项来绑定接口的物理层的组播mac地址等，好像是会通知网卡接收组播的。
```
	struct sockaddr sa;

	sa.sa_family = AF_PACKET;
	strncpy(sa.sa_data, "eth0", sizeof(sa.sa_data));

	if (bind(sock, &sa, sizeof sa)) {
    // error
	}
```

# 发送链路层的包
在buf中构造好以太网头，vlan头，ip地址，udp等（需要自己算校验）就可以直接发送了。
好像这个socket也有一种模式可以让内核帮你自动插入以太网的mad地址头部的吧。
```text
struct sockaddr addr;
strcpy(addr.sa_data, "eth0");
len = sendto(sock, buf, buf_len, 0, &addr, sizeof(addr));
```

# 抓包/接收链路层的包
如果是用于抓包的话， 这个PACKET socket应该是可以指定一个关联的Berkeley Packet Filter
过滤器的，内核里面里面是执行bpf代码过滤出目的包才会发给这个socket。bpf好像很火啊，
很多模块都会使用不单单是这个抓包过滤用到了。
tcpdump/libpcap库的抓包的过滤语法，就是编译为bpf代码放到内核里面。tcpdump应该也是
用这个packet socket来做的吧，libpcap应该是对这个做了进一步的封装吧，可以libpcap的跨平台特性更好一些吧。

```text
       The Linux kernel source tree.  /Documentation/networking/filter.txt
       describes how to apply Berkeley Packet Filters to packet sockets.
       /tools/testing/selftests/net/psock_tpacket.c contains example source
       code for all available versions of PACKET_RX_RING and PACKET_TX_RING.
```

内核文档给出的设置filter的例子
```c
struct sock_fprog bpf = {
	.len = ARRAY_SIZE(code),
	.filter = code,
};

sock = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
if (sock < 0)
	/* ... bail out ... */

ret = setsockopt(sock, SOL_SOCKET, SO_ATTACH_FILTER, &bpf, sizeof(bpf));
if (ret < 0)
	/* ... bail out ... */

/* ... */
close(sock);
```




# PACKET socket内核对应的代码在
net/packet/af_packet.c


这篇文章做了很多介绍
[Inside the Linux Packet Filter, Part II](https://www.linuxjournal.com/article/5617)

# bpf过滤器的实现
core/filter.c
```text
The main filter implementation resides in core/filter.c, whereas the SO_ATTACH/DETACH_FILTER ioctls are dealt with in net/core/sock.c. The filter initially is attached to a socket via the sk_attach_filter() function, that copies it from user space to kernel space and runs an integrity check on it (sk_chk_filter()). 
```
