```text
看上去现在tc的还是htb加fq_codel的组合为主流。看一下介绍和源码，还有别人的例子。

手册：
http://man7.org/linux/man-pages/man8/tc.8.html
http://man7.org/linux/man-pages/man8/tc-ematch.8.html  filter分类支持简单的表达式，cmp/and/or/字节检查/ipset/xtables等等的
http://man7.org/linux/man-pages/man8/tc-bpf.8.html     filter也是支持bpf的自定义扩展的
http://man7.org/linux/man-pages/man8/tc-htb.8.html
http://man7.org/linux/man-pages/man8/tc-fq_codel.8.html

介绍：
https://www.bufferbloat.net/projects/codel/wiki/Best_practices_for_benchmarking_Codel_and_FQ_Codel/
http://luxik.cdi.cz/~devik/qos/htb/manual/userg.htm
http://tldp.org/HOWTO/Traffic-Control-HOWTO/classful-qdiscs.html#qc-htb
https://www.linuxjournal.com/article/7562
https://www.cnblogs.com/acool/p/7779159.html
https://www.docum.org/docum.org/tests/htb/burst/

源码：
https://elixir.bootlin.com/linux/latest/source/net/sched/em_meta.c
https://elixir.bootlin.com/linux/latest/source/net/sched/sch_tbf.c
https://elixir.bootlin.com/linux/latest/source/net/sched/sch_htb.c
https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/tree/tc/sch_tbf.c
https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/tree/tc/q_htb.c

openwrt的例子：
https://github.com/tohojo/sqm-scripts/blob/master/src/simple.qos

man tc
man tc-htb
man tc-tbf
man tc-fc_codel


centos 7 中 基本表达式中支持的meta
/usr/sbin/tc filter add dev eth1 basic match 'meta(list)'
--------------------------------------------------------
  ID               Type       Description
--------------------------------------------------------
Generic:
  random           INT        Random value (32 bit)
  loadavg_1        INT        Load average in last minute
  loadavg_5        INT        Load average in last 5 minutes
  loadavg_15       INT        Load average in last 15 minutes

Interfaces:
  dev              INT,VAR    Device the packet is on

Packet attributes:
  priority         INT        Priority of packet
  protocol         INT        Link layer protocol
  pkt_type         INT        Packet type (uni|multi|broad|...)cast
  pkt_len          INT        Length of packet
  data_len         INT        Length of data in packet
  mac_len          INT        Length of link layer header

Netfilter:
  nf_mark          INT        Netfilter mark
  fwmark           INT        Alias for nf_mark

Traffic Control:
  tc_index         INT        TC Index

Routing:
  rt_classid       INT        Routing ClassID (cls_route)
  rt_iif           INT        Incoming interface index
  vlan             INT        Vlan tag

Sockets:
  sk_family        INT        Address family
  sk_state         INT        State
  sk_reuse         INT        Reuse Flag
  sk_bind_if       INT,VAR    Bound interface
  sk_refcnt        INT        Reference counter
  sk_shutdown      INT        Shutdown mask
  sk_proto         INT        Protocol
  sk_type          INT        Type
  sk_rcvbuf        INT        Receive buffer size
  sk_rmem          INT        RMEM
  sk_wmem          INT        WMEM
  sk_omem          INT        OMEM
  sk_wmem_queue    INT        WMEM queue
  sk_snd_queue     INT        Send queue length
  sk_rcv_queue     INT        Receive queue length
  sk_err_queue     INT        Error queue length
  sk_fwd_alloc     INT        Forward allocations
  sk_sndbuf        INT        Send buffer size
--------------------------------------------------------




概念：qdisc 是队列，下面可以包含多个class，每个class就是分类了，可以多个树形结构的级别，
filter是就是分类规则了，每个class可以设置不同的filter来分类决定包被分到那个子级别的class里面去。
可以先看一下 tc的man page里面THEORY OF OPERATION   小节的说明


tc qdisc show dev eth1
tc -s qdisc show dev eth1
tc -s -d class show dev eth1
tc --g -s class show dev eth1
tc -s filter show dev eth1

tc qdisc del dev eth1 root
tc qdisc add dev eth1 root handle 1: htb default 13  r2q 10
tc class add dev eth1 parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit burst 3000 cburst 3000
tc class add dev eth1 parent 1:1 classid 1:11 htb rate 100mbit ceil 1000mbit burst 3000 cburst 3000 prio 1
tc class add dev eth1 parent 1:1 classid 1:12 htb rate 500mbit ceil 1000mbit burst 3000 cburst 3000 prio 2
tc class add dev eth1 parent 1:1 classid 1:13 htb rate 50mbit ceil 50mbit burst 3000 cburst 3000 prio 3
tc class add dev eth1 parent 1:13 classid 1:14 htb rate 20mbit ceil 20mbit burst 3000 cburst 3000 prio 1
tc class add dev eth1 parent 1:13 classid 1:15 htb rate 30mbit ceil 30mbit burst 3000 cburst 3000 prio 1

tc qdisc add dev eth1 parent 1:11 handle 110: fq_codel limit 1024 ecn
tc qdisc add dev eth1 parent 1:12 handle 120: fq_codel limit 1024 ecn
tc qdisc add dev eth1 parent 1:14 handle 140: fq_codel limit 1024 ecn
tc qdisc add dev eth1 parent 1:15 handle 150: fq_codel limit 1024 noecn

# tc filter add dev eth1 basic match 'meta(pkt_len gt 0)' flowid 1:13
# tc filter add dev eth1 protocol ip basic match 'meta(pkt_len gt 0)'  flowid 1:13
# tc filter add dev eth1 basic match 'meta(dev eq 8)' flowid 1:13
# tc filter add dev eth1 basic match 'meta(rt_iif gt 0)' flowid 1:14
tc filter add dev eth1 basic match 'meta(fwmark gt 24)' flowid 1:15
tc filter add dev eth1 parent 1:13 handle 256 fw flowid 1:14
tc filter add dev eth1 parent 1:13 handle 257 fw flowid 1:1


tc 本身会自动计算burst和cburst，设置为
/* compute minimal allowed burst from rate; mtu is added here to make
   sute that buffer is larger than mtu and to have some safeguard space */
if (!buffer)
  buffer = rate64 / get_hz() + mtu;
if (!cbuffer)
  cbuffer = ceil64 / get_hz() + mtu;

/ # cat /proc/net/psched
000003e8 00000040 000f4240 3b9aca00

但现在的系统tc从/proc/net/psched获取的HZ都是很大的，跟网卡NAPI等接口有关？
所以基本算出来都是默认的mtu 1600byte，但HTB这些算法，现在的都是把包长度表示成
纳秒的时间刻度的，所以转换完再转换回来好像数值有点不一样，经常看到比MTU要小的。
本来HZ比较小的时候，这个burst和cburst是要设置的很大的比较十几KB的，这样才能发
完带宽。但好像现在内核API变化了，这个跟系统时钟周期没关系了，


外网上行接口不开ecn，内网可以开ecn
```
