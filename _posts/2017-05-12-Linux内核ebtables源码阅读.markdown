The ebtables program is a filtering tool for a Linux-based bridging firewall. 
  It enables transparent filtering of network traffic passing through a Linux bridge. 
**只能应用于linux bridge桥设备上面，这个是硬伤，比如你想应用于某个物理网卡，那首先要配置linux桥设备，然后把物理网卡作为桥设备的子端口添加上去才行**


最好先man ebtables或者 看看 https://linux.die.net/man/8/ebtables 的说明，
理解table  chain rule 和 match  target等概念。应该比iptables简单的多吧。


网络包的钩子和规则的遍历执行
----------------------------

ebtable_broute.c
ebtable_filter.c
```c
static int __init ebtable_broute_init(void)
{
	int ret;

	ret = register_pernet_subsys(&broute_net_ops);
	if (ret < 0)
		return ret;
	/* see br_input.c */
	RCU_INIT_POINTER(br_should_route_hook,              //  broute hook
			   (br_should_route_hook_t *)ebt_broute);
	return 0;
}



static int __init ebtable_filter_init(void)
{
	int ret;

	ret = register_pernet_subsys(&frame_filter_net_ops);
	if (ret < 0)
		return ret;
	ret = nf_register_hooks(ebt_ops_filter, ARRAY_SIZE(ebt_ops_filter));
	if (ret < 0)
		unregister_pernet_subsys(&frame_filter_net_ops);
	return ret;
}

static struct nf_hook_ops ebt_ops_filter[] __read_mostly = {
	{
		.hook		= ebt_in_hook,
		.pf		= NFPROTO_BRIDGE,
		.hooknum	= NF_BR_LOCAL_IN,
		.priority	= NF_BR_PRI_FILTER_BRIDGED,
	},
	{
		.hook		= ebt_in_hook,
		.pf		= NFPROTO_BRIDGE,
		.hooknum	= NF_BR_FORWARD,
		.priority	= NF_BR_PRI_FILTER_BRIDGED,
	},
	{
		.hook		= ebt_out_hook,
		.pf		= NFPROTO_BRIDGE,
		.hooknum	= NF_BR_LOCAL_OUT,
		.priority	= NF_BR_PRI_FILTER_OTHER,
	},
};

```

所有的网络包都通过这几个钩子函数进来，然后都会执行到ebtables.c 里面的ebt_do_table来进行最终的处理
```text
ebt_broute   // br_input.c 里面全局的 br_should_route_hook 调用 
ebt_out_hook
ebt_in_hook
     ebt_do_table  // 这里面会遍历匹配table里面的规则来执行对应target
          ebt_do_match  // 执行各种匹配
             m->u.match->match(skb, par)    // 执行各种预先注册的match方法 
          struct ebt_entry_target *t;
          verdict = t->u.target->target(skb, &acpar);   // 调用target 

各种协议预先注册规则 match（struct xt_match） 和target（struct xt_target）
xt_register_match(&ebt_arp_mt_reg);
xt_register_target(&ebt_arpreply_tg_reg);
```      

各个模块初始化时会调用ebt_register_table 来创建 nat ，filter ， broute等 table的。
不过ebt_broute 这个钩子是通过桥接口的rx_handler br_handle_frame来进行的，好像只会
在brigde设备上才会预先通过br_add_if配置这个hook，，所以只有从桥设备的端口进来的包才能调用到这些hook函数。
**再仔细看了一下ebt_out_hook和ebt_in_hook也都只能从bridge接口进来的包才会调用**
**原来ebtables就是只能在Linux bridge设备上面使用的，难怪代码都是放在bridge目录下。这样ebtables的使用场合就很有限了**
**要想用ebtables操作某个物理网卡上面的进来的包，首先要创建linux 桥设备，然后把这个物理网卡作为桥设备的子端口添加上去才行**
**这样使用就不太方便了。我之前还以为这个ebtables可以直接过来所有的物理网卡上面的包呢**


TARGET 
------

- arpreply 
   这个target比较有意思，可以自己构造回复ARP请求。
ebt_arpreply.c
```c
static struct xt_target ebt_arpreply_tg_reg __read_mostly = {
	.name		= "arpreply",
	.revision	= 0,
	.family		= NFPROTO_BRIDGE,
	.table		= "nat",
	.hooks		= (1 << NF_BR_NUMHOOKS) | (1 << NF_BR_PRE_ROUTING),
	.target		= ebt_arpreply_tg,
	.checkentry	= ebt_arpreply_tg_check,
	.targetsize	= sizeof(struct ebt_arpreply_info),
	.me		= THIS_MODULE,
};

static int __init ebt_arpreply_init(void)
{
	return xt_register_target(&ebt_arpreply_tg_reg);
}
```

- limit 
    另外一个的match也比较有意思，可以限制每秒发包的个数
- dnat和snat
    这两个target可以修改数据包的目的和源MAC
- ulog 
    可以把网络包转发到userspace的netlink去处理。通过nf_log_packet函数来实现的。
