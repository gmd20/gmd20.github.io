# 用ping检测MTU多大可以成功
```text
ping-l 1432 -f -n 1  8.8.8.8 
```

# 以太网的标准的MTU是1500

# centos 修改接口的MTU
```text
echo 1470 > /sys/class/net/eth0/mtu
```

# 设置linux 的 "Path MTU Discovery"， 可以设置系统/socket自动探测MTU
tcp、ip里面有标准怎么探测这个路径上的MTU的，系统应该会自动根据icmp错误报告记录下路径上的MTU。
```text
echo 1 > /proc/sys/net/ipv4/ip_no_pmtu_disc
```
/proc/sys/net/ipv4/ip_no_pmtu_disc 的设置支持下面这几个值
```c
/* IP_MTU_DISCOVER values */
#define IP_PMTUDISC_DONT		0	/* Never send DF frames */
#define IP_PMTUDISC_WANT		1	/* Use per route hints	*/
#define IP_PMTUDISC_DO			2	/* Always DF		*/
#define IP_PMTUDISC_PROBE		3 /* Ignore dst pmtu      */
```
参考 http://man7.org/linux/man-pages/man7/ip.7.html


# 探测网络节点的MTU
```text
tracepath baidu.com
```


# TCP 的MSS 
TCP的建立的时候在syn包中会设置一个MSS选项，用于通知对端最大的MSS大小。MSS = MTU - header_size
iptables 中有个TCPMSS模块可以修改syn中mss，这样在中间设备上就可以修改MSS为新的值。交换机路由器上面好像一般都可以配置规则修改MSS。
应该是有些sb运营商会丢掉 "ICMP Fragmentation Needed" 这个提示需要分片的icmp报告，导致"Path MTU Discovery"探测没有能够探测出路径上
的实际MTU大小，这样传输层tcp里面就会用了比较大的MSS，但实际路径上的MTU很小，这样tcp大包就过不来了。中间强制修改了MSS会强制双方通讯
的包大小按照设置的来。发现有的运营商的pppoe拨号线路会这有这个问题，需要强制设置MSS才行。
```text
TCPMSS

This target allows to alter the MSS value of TCP SYN packets, to control the maximum size for that connection (usually limiting it to your outgoing interface's MTU minus 40 for IPv4 or 60 for IPv6, respectively). Of course, it can only be used in conjunction with -p tcp.
This target is used to overcome criminally braindead ISPs or servers which block "ICMP Fragmentation Needed" or "ICMPv6 Packet Too Big" packets. The symptoms of this problem are that everything works fine from your Linux firewall/router, but machines behind it can never exchange large packets:

1.
Web browsers connect, then hang with no data received.
2.
Small mail works fine, but large emails hang.
3.
ssh works fine, but scp hangs after initial handshaking.
Workaround: activate this option and add a rule to your firewall configuration like:


 iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN
             -j TCPMSS --clamp-mss-to-pmtu
--set-mss value
Explicitly sets MSS option to specified value. If the MSS of the packet is already lower than value, it will not be increased (from Linux 2.6.25 onwards) to avoid more problems with hosts relying on a proper MSS.
--clamp-mss-to-pmtu
Automatically clamp MSS value to (path_MTU - 40 for IPv4; -60 for IPv6). This may not function as desired where asymmetric routes with differing path MTU exist --- the kernel uses the path MTU which it would use to send packets from itself to the source and destination IP addresses. Prior to Linux 2.6.25, only the path MTU to the destination IP address was considered by this option; subsequent kernels also consider the path MTU to the source IP address.
These options are mutually exclusive.  
```

# 网上的这个文章好像讲的还挺详细的
[MTU & MSS 详解记录](http://blog.51cto.com/infotech/123859)
