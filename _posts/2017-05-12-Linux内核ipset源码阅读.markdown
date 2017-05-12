我之前还在想怎么自己在内核管理大量的ip和mac地址，然后自己做hash匹配之类的。不过linux内核里面有个ipset真是太方便了。
联合iptables使用的话，用ipset 来管理大量的ip 端口还有mac地址这些，这样就可以避免创建大量条目的 iptables规则了。
只用创建一条规则，然后set match and SET target of iptables 就可以了。
这样在有的场合，比如要封禁大量ip的时候，非常有用，应该可以大量提高系统性能。因为iptables如果上千的规则条目性能就下降的很厉害了。随便最新的实现有所
改进，但还是ipset这种方式好吧。

支持的特性可以参考官网
http://ipset.netfilter.org/features.html

对应的内核源码在这里
linux-4.11\net\netfilter\ipset 

大概看了一下，hash表也是使用高性能jhash 来实现的，跟 conntrack那个用的一样的。也有RCU优化了。用起来肯定比自己去实现一套要好了。
代码风格 使用了“X Macro”技巧的代码自动生成， 比如hash table的实现主要在  ip_set_hash_gen.h里面

用户空间的ipset命令通过 libipset.so 这个库和内核通讯，内核实现了netlink接口了
```c

static const struct nfnl_callback ip_set_netlink_subsys_cb[IPSET_MSG_MAX] = {
	[IPSET_CMD_NONE]	= {
		.call		= ip_set_none,
		.attr_count	= IPSET_ATTR_CMD_MAX,
	},
	[IPSET_CMD_CREATE]	= {
		.call		= ip_set_create,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_create_policy,
	},
	[IPSET_CMD_DESTROY]	= {
		.call		= ip_set_destroy,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_setname_policy,
	},
	[IPSET_CMD_FLUSH]	= {
		.call		= ip_set_flush,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_setname_policy,
	},
	[IPSET_CMD_RENAME]	= {
		.call		= ip_set_rename,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_setname2_policy,
	},
	[IPSET_CMD_SWAP]	= {
		.call		= ip_set_swap,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_setname2_policy,
	},
	[IPSET_CMD_LIST]	= {
		.call		= ip_set_dump,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_setname_policy,
	},
	[IPSET_CMD_SAVE]	= {
		.call		= ip_set_dump,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_setname_policy,
	},
	[IPSET_CMD_ADD]	= {
		.call		= ip_set_uadd,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_adt_policy,
	},
	[IPSET_CMD_DEL]	= {
		.call		= ip_set_udel,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_adt_policy,
	},
	[IPSET_CMD_TEST]	= {
		.call		= ip_set_utest,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_adt_policy,
	},
	[IPSET_CMD_HEADER]	= {
		.call		= ip_set_header,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_setname_policy,
	},
	[IPSET_CMD_TYPE]	= {
		.call		= ip_set_type,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_type_policy,
	},
	[IPSET_CMD_PROTOCOL]	= {
		.call		= ip_set_protocol,
		.attr_count	= IPSET_ATTR_CMD_MAX,
		.policy		= ip_set_protocol_policy,
	},
};

static struct nfnetlink_subsystem ip_set_netlink_subsys __read_mostly = {
	.name		= "ip_set",
	.subsys_id	= NFNL_SUBSYS_IPSET,
	.cb_count	= IPSET_MSG_MAX,
	.cb		= ip_set_netlink_subsys_cb,
};
```
