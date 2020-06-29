# CentOS的虚拟机设置使用了多个网卡，发现只有一个网卡的外部网络通讯是正常的

# 用ping命令测试排查，结合tcpdump抓包排查，发现网络包是可以出去，icmp reply也回到本机了但被本机丢掉了，用dropwatch命令可以但在本机的ip_rcv_finish函数里面被丢包了
```text
ping 163.com -I enp0s3
ping 163.com -I enp0s8
```

# 检查路由没有问题， 发现只有 metric值最小的网卡通讯是可以正常的
```text
# ip route
default via 172.31.22.1 dev enp0s8 proto dhcp metric 100 
default via 192.168.56.1 dev enp0s9 proto static metric 101 
default via 10.0.2.2 dev enp0s3 proto dhcp metric 102 
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 102 
172.31.20.0/24 dev enp0s10 proto kernel scope link src 172.31.20.200 metric 103 
172.31.22.0/24 dev enp0s8 proto kernel scope link src 172.31.22.3 metric 100 
192.168.56.0/24 dev enp0s9 proto kernel scope link src 192.168.56.107 metric 101 
```

# 看 ip_rcv_finish 代码丢包的原因感觉应该还是路由有关
```c
static int ip_rcv_finish_core(struct net *net, struct sock *sk,
			      struct sk_buff *skb, struct net_device *dev,
			      const struct sk_buff *hint)
{
	const struct iphdr *iph = ip_hdr(skb);
	int (*edemux)(struct sk_buff *skb);
	struct rtable *rt;
	int err;

	if (ip_can_use_hint(skb, iph, hint)) {
		err = ip_route_use_hint(skb, iph->daddr, iph->saddr, iph->tos,
					dev, hint);
		if (unlikely(err))
			goto drop_error;
	}

	if (net->ipv4.sysctl_ip_early_demux &&
	    !skb_dst(skb) &&
	    !skb->sk &&
	    !ip_is_fragment(iph)) {
		const struct net_protocol *ipprot;
		int protocol = iph->protocol;

		ipprot = rcu_dereference(inet_protos[protocol]);
		if (ipprot && (edemux = READ_ONCE(ipprot->early_demux))) {
			err = INDIRECT_CALL_2(edemux, tcp_v4_early_demux,
					      udp_v4_early_demux, skb);
			if (unlikely(err))
				goto drop_error;
			/* must reload iph, skb->head might have changed */
			iph = ip_hdr(skb);
		}
	}

	/*
	 *	Initialise the virtual path cache for the packet. It describes
	 *	how the packet travels inside Linux networking.
	 */
	if (!skb_valid_dst(skb)) {
		err = ip_route_input_noref(skb, iph->daddr, iph->saddr,   // 不同的版本代码稍微有区别，最可能还是这个地方查路由失败了
					   iph->tos, dev);
		if (unlikely(err))
			goto drop_error;
	}

#ifdef CONFIG_IP_ROUTE_CLASSID
	if (unlikely(skb_dst(skb)->tclassid)) {
		struct ip_rt_acct *st = this_cpu_ptr(ip_rt_acct);
		u32 idx = skb_dst(skb)->tclassid;
		st[idx&0xFF].o_packets++;
		st[idx&0xFF].o_bytes += skb->len;
		st[(idx>>16)&0xFF].i_packets++;
		st[(idx>>16)&0xFF].i_bytes += skb->len;
	}
#endif

	if (iph->ihl > 5 && ip_rcv_options(skb, dev))
		goto drop;

	rt = skb_rtable(skb);
	if (rt->rt_type == RTN_MULTICAST) {
		__IP_UPD_PO_STATS(net, IPSTATS_MIB_INMCAST, skb->len);
	} else if (rt->rt_type == RTN_BROADCAST) {
		__IP_UPD_PO_STATS(net, IPSTATS_MIB_INBCAST, skb->len);
	} else if (skb->pkt_type == PACKET_BROADCAST ||
		   skb->pkt_type == PACKET_MULTICAST) {
		struct in_device *in_dev = __in_dev_get_rcu(dev);

		/* RFC 1122 3.3.6:
		 *
		 *   When a host sends a datagram to a link-layer broadcast
		 *   address, the IP destination address MUST be a legal IP
		 *   broadcast or IP multicast address.
		 *
		 *   A host SHOULD silently discard a datagram that is received
		 *   via a link-layer broadcast (see Section 2.4) but does not
		 *   specify an IP multicast or broadcast destination address.
		 *
		 * This doesn't explicitly say L2 *broadcast*, but broadcast is
		 * in a way a form of multicast and the most common use case for
		 * this is 802.11 protecting against cross-station spoofing (the
		 * so-called "hole-196" attack) so do it for both.
		 */
		if (in_dev &&
		    IN_DEV_ORCONF(in_dev, DROP_UNICAST_IN_L2_MULTICAST))
			goto drop;
	}

	return NET_RX_SUCCESS;

drop:
	kfree_skb(skb);
	return NET_RX_DROP;

drop_error:
	if (err == -EXDEV)
		__NET_INC_STATS(net, LINUX_MIB_IPRPFILTER);
	goto drop;
}

static int ip_rcv_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	struct net_device *dev = skb->dev;
	int ret;

	/* if ingress device is enslaved to an L3 master device pass the
	 * skb to its handler for processing
	 */
	skb = l3mdev_ip_rcv(skb);
	if (!skb)
		return NET_RX_SUCCESS;

	ret = ip_rcv_finish_core(net, sk, skb, dev, NULL);
	if (ret != NET_RX_DROP)
		ret = dst_input(skb);
	return ret;
}

```

#  用ip route  get 命令模拟查询一下路由
```text
# ip route get 10.0.2.15 from 180.76.76.76 iif enp0s3
RTNETLINK answers: Invalid cross-device link

# ip route get 10.0.2.15 from 10.0.2.2 iif enp0s3
local 10.0.2.15 from 10.0.2.2 dev lo 
    cache <local> iif enp0s3 

# ip route get 172.31.22.3 from 180.76.76.76  iif enp0s8
local 172.31.22.3 from 180.76.76.76 dev lo 
    cache <local> iif enp0s8
```
模拟路由的结果跟实测的结果是一样的，关键就是这个“Invalid cross-device link” 错误，即使第二个网卡上外网不通丢包的原因了。


#  在网上搜索了一下 RTNETLINK answers: Invalid cross-device link
找到 这个网页 https://bbs.archlinux.org/viewtopic.php?id=199573提到rp_filter   
检查一下确实发现centos8的这个 rp_filter默认是打开的, 关闭一下第二个网卡的网络马上正常了。
```text
# cat /proc/sys/net/ipv4/conf/*/rp_filter
1
0
0
0
0
0
0
# cat /proc/sys/net/ipv4/conf/all/rp_filter
1

# echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter
```


# rp_filter 到是什么东西呢
https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html
```text
rp_filter - INTEGER
    0 - No source validation.
    1 - Strict mode as defined in RFC3704 Strict Reverse Path Each incoming packet is tested against the FIB and if the interface is not the best reverse path the packet check will fail. By default failed packets are discarded.
    2 - Loose mode as defined in RFC3704 Loose Reverse Path Each incoming packet’s source address is also tested against the FIB and if the source address is not reachable via any interface the packet check will fail.
Current recommended practice in RFC3704 is to enable strict mode to prevent IP spoofing from DDos attacks. If using asymmetric routing or other complicated routing, then loose mode is recommended.

    The max value from conf/{all,interface}/rp_filter is used when doing source validation on the {interface}.

Default value is 0. Note that some distributions enable it in startup scripts.
```
这个中文翻译叫做 “反向路由检查”吧， 就是主机收到包，会反向的查一下路由看看反向的是不是同一个路径，如果不是同一个路径就直接丢掉了。 这个在这个虚拟机环境，metric最小的默认路由
只有一个，反查的结果肯定是从第一个网卡出去了，所以第二个网卡收上来的包肯定是会被直接丢掉的。 所以用了 多网卡（multiple NIC）的情况要是期待 多个网口都能和外网通讯，这个rp_filter
是不能使用  “strict mode” 严格模式的。

# centos 7 一直都是没有问题的，不知道centos 8 默认设置怎么调整了这个
测试时不同NetworkManager程序版本，还是哪些脚本之类的错误的设置了这个东西，看 /usr/share/doc/NetworkManager/NEWS里面日志有好多rp_filter的说明。




