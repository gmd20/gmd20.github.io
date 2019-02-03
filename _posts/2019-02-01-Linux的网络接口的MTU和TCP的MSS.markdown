# 用ping检测MTU多大可以成功
```text
ping  -s 1450  8.8.8.8
```

# 以太网的标准的MTU是1500

# centos 修改接口的MTU
```text
echo 1470 > /sys/class/net/eth0/mtu
```

# 设置linux 的 "Path MTU Discovery"， 可以设置系统/socket自动探测MTU
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
iptables 中有个TCPMSS模块可以修改syn中mss，这样在中间设备上就可以修改MSS为新的值。交换机路由器上面好像一般都可以配置规则修改MSS，错误的修改应该会导致两端的通讯出问题。
