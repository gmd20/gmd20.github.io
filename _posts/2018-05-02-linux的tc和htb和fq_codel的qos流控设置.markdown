```text
看上去现在tc的还是htb加fq_codel的组合为主流。看一下介绍和源码，还有别人的例子。

手册：
http://man7.org/linux/man-pages/man8/tc.8.html
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

tc qdisc show dev eth1
tc -s qdisc show dev eth1
tc -s -d qdisc show dev eth1
tc -s -d class show dev eth1

tc qdisc del dev eth1 root
tc qdisc add dev eth1 root handle 1: htb default 13
tc class add dev eth1 parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit burst 3000 cburst 3000
tc class add dev eth1 parent 1:1 classid 1:11 htb rate 100mbit ceil 1000mbit burst 3000 cburst 3000 prio 1
tc class add dev eth1 parent 1:1 classid 1:12 htb rate 200mbit ceil 1000mbit burst 3000 cburst 3000 prio 2
tc class add dev eth1 parent 1:1 classid 1:13 htb rate 700mbit ceil 1000mbit burst 3000 cburst 3000 prio 3

tc qdisc add dev eth1 parent 1:11 handle 110: fq_codel limit 1024 ecn
tc qdisc add dev eth1 parent 1:12 handle 120: fq_codel limit 1024 noecn
tc qdisc add dev eth1 parent 1:13 handle 130: fq_codel limit 1024 noecn

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
