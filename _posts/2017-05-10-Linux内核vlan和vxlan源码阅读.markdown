vlan (802.1q)
=====

通常的vlan是指IEEE 802.1q标准定义的virtual LANs, 就是在以太网头部插入一个vlan
tag来划分子网的技术。
linux内核的vlan相关代码都在源码的linux-4.11\net\8021q 这个目录下面。


操作skb的vlan的辅助函数
vlan_insert_tag
skb_vlan_untag
skb_vlan_pop
skb_vlan_push

接收
----
接收上来的包明显要经过vlan_do_receive函数来处理，用这个函数一搜索就看到包怎么从
协议栈层传上来了，但这个从网络协议栈上来的时候，之前就在__netif_receive_skb_core
里面做了untag处理了，这点最初没注意到找了好一会。

http://elixir.free-electrons.com/linux/latest/source/net/core/dev.c#L4152
```text
static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)
{
	if (skb->protocol == cpu_to_be16(ETH_P_8021Q) ||
    skb->protocol == cpu_to_be16(ETH_P_8021AD)) {
		skb = skb_vlan_untag(skb);    // 读出vlan id，去掉 vlan tag，
		if (unlikely(!skb))
			goto out;
	}

 //中间处理各种注册的协议和抓包等。

	if (skb_vlan_tag_present(skb)) {
		if (pt_prev) {
			ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = NULL;
		}
		if (vlan_do_receive(&skb))   // 根据vlan id设置skb的dev 为vlan dev等
			goto another_round;        // 重新进入协议栈，再遍历一次各种注册的协议就会运行ip协议层函数处理了
		else if (unlikely(!skb))
			goto out;
	}

}
```

从代码里面看调用这个vlan_do_receive 应该是在bridge的rx_handler hook点的前面、。



发送
----
发送很容易看，找vlan net device 对应的 ops 函数就可以了。
vlan_dev.c
vlan_dev_hard_start_xmit


不相关的"vlan"
-------------------------
这个vlan完全和802.1q没有任何管理，就是能划分流量到多个interface上面去而已吧。
这里的介绍不错 http://hicu.be/macvlan-vs-ipvlan
http://hicu.be/bridge-vs-macvlan

Macvlan （linux-4.11\drivers\net\macvlan.c）在一个物理网卡上面虚拟出支持多个
         虚拟网卡（多个mac地址），看起来就像你的机器有几块网卡一样。
         有点类似 bridge ， 但应该理念不一样了。
         docker都有支持macvlanx网络划分，把创建出来的设备分给不同虚拟机。
         https://docs.docker.com/engine/userguide/networking/get-started-macvlan/

ipvlan （linux-4.11\drivers\net\ipvlan)
         https://github.com/torvalds/linux/blob/master/Documentation/networking/ipvlan.txt
        跟 macvlan很类似的技术，但所有的接口都公用底层的一个mac。这个貌似跟设置
        多个ip到一个接口上面不同的地方，就是ipvlan有多个interface。这样就可以
        把创建出来的interface 分给不同的虚拟机来用吧。


https://en.wikipedia.org/wiki/IEEE_802.1Q
网上其他人写的linux内核的vlan相关的文章
http://www.cnblogs.com/zmkeil/archive/2013/04/18/3029339.htm\
http://blog.csdn.net/bc_vnetwork/article/details/51787473



openvswitch的vlan
================
http://docs.openvswitch.org/en/latest/howto/vlan/
http://docs.openvswitch.org/en/latest/faq/vlan/

openvswitch的action 里面有pop_vlan push_vlan可以把网络包的vlan tag给修改了吧


ebtables可以过滤vlan tag的包，进行处理等？
========================================
ebtables -t broute -A BROUTING -i eth0 -p 802_1Q -j DROP


linux vlan的连接交换机的“vlan trunk mode”的文件
===============================================
还有个一个不太理解的问题，因为linux要支持一个vlan的话
都会创建要给对应的虚拟net device，比如 eth0.100这种。
但如果eth0连的交换机的trunk接口的时候，要支持比较多的vlan id
比如4000个vlan， 难道要对应linux系统自己创建4000个vlan设备才行？
这种情形可能把这个linux作为nat网关，给交换机后面的多台设备用时可能
会碰到。如果按照常规的一个vlan id对应要给eth0.100虚拟设备这种方法
感觉太耗资源了。
想到一个办法，只能自己管理vlan id ，接收上来的可以自己用ovs或者ebtables这些
辅助删掉然后再按照所有包都是untaged的来处理。 但linux要回复的时候，又需要自己
把vlan tag给补回去。 这个自己写一个模块hook发送和接收的处理过程。感觉还是非常
麻烦的，需要自己维护vlan id和mac的映射表。
就有没有什么其他简便的方法，谁知道的话，可以告诉我。


vxlan
=====
linux-4.11\drivers\net\vxlan.c

vxlan是vlan的改进，把二层的以太网包封装在udp层发送出去
这个的实现代码比vlan简单好多了，就是创建udp连接，udp收上来的包
调用 vxlan_rcv和 vxlan_gro_receive进行处理，然后调用gro_cells_receive和
eth_gro_receive 投递到内核协议栈。 发送更简单，vxlan_xmit函数里面封装成udp
就直接发送出去了。
```c
static const struct net_device_ops vxlan_netdev_raw_ops = {
	.ndo_init		= vxlan_init,
	.ndo_uninit		= vxlan_uninit,
	.ndo_open		= vxlan_open,
	.ndo_stop		= vxlan_stop,
	.ndo_start_xmit		= vxlan_xmit,
	.ndo_get_stats64	= ip_tunnel_get_stats64,
	.ndo_change_mtu		= vxlan_change_mtu,
	.ndo_fill_metadata_dst	= vxlan_fill_metadata_dst,
};
```


