helper是一些连接跟踪的辅助，可以认为是dpi的功能，会深入解析协议细节
==================================================================

比如 nf_conntrack_pptp.c 就是PPTP协议的dpi解析，解析PPTP里面的call_id 和 其他会话建立过程等等，
然后nf_nat_pptp可以替换pptp包里面call id，避免内网多个用户使用同一个call-id冲突，这样NAT内网的用户也正常PPTP拨号。


这些辅助模块，主要通过 nf_conntrack_helper_register 注册内核helper模块， conntrack在合适的时机调用helper的解析函数。
```c
static struct nf_conntrack_helper pptp __read_mostly = {
	.name			= "pptp",
	.me			= THIS_MODULE,
	.tuple.src.l3num	= AF_INET,
	.tuple.src.u.tcp.port	= cpu_to_be16(PPTP_CONTROL_PORT),
	.tuple.dst.protonum	= IPPROTO_TCP,
	.help			= conntrack_pptp_help,
	.destroy		= pptp_destroy_siblings,
	.expect_policy		= &pptp_exp_policy,
};

static int __init nf_conntrack_pptp_init(void)
{
	NF_CT_HELPER_BUILD_BUG_ON(sizeof(struct nf_ct_pptp_master));

	return nf_conntrack_helper_register(&pptp);
}

static void __exit nf_conntrack_pptp_fini(void)
{
	nf_conntrack_helper_unregister(&pptp);
}
```

PPTP 主要逻辑
=============
```text
  conntrack_pptp_help
      pptp_outbound_pkt
      pptp_inbound_pkt
         exp_gre  添加 预期的gre连接的expect表
             nf_ct_gre_keymap_add

nf_conntrack_proto_gre.c
    gre_pkt_to_tuple
      gre_keymap_lookup   查找gre流的 expect表，关联到 pptp的tcp 1723连接的信息
```

PPTP的协议首先是内外发起到服务器tcp 1723端口的PPTP连接，协商好了后续外网服务器会新建GRE隧道。GRE隧道的流从外面进来时会携带协商好的call id，helper模块看到GRE连接会根据expect把他关联到 tcp 1723 这个连接跟踪ct结构，
这个tcp 1723是内网首先发起的有内网nat用户ip，这个外网发起的GRRE才能被正确的NAT转发到同一个用户。 如果没有helper这个辅助和expect的关系，这个GRE外面进来的连接会被直接丢弃，导致PPTP拨号失败。





         
nf_conntrack_helper.c
=====================
老版本的内核会在conntrack初始化时init_conntrack -> __nf_ct_try_assign_helper 来自动加载helper模块，
nf_conntrack模块有个“nf_conntrack_helper”命令行参数控制是否自动加载helper模块，有些协议的helper可能会导致安全问题所以默认sysctl_auto_assign_helper是没有打开的。
所以要想启用这个PPTP的helper模块， 要注意一下nf_conntrack_helper这个参数设置
```text
modprobe nf_conntrack hashsize=32768 acct=1 nf_conntrack_helper=1
modprobe nf_conntrack_pptp
modprobe nf_nat_pptp
echo 1 > /proc/sys/net/netfilter/nf_conntrack_helper
```

最新的内核好像，这个自动加载helper的都没有了，需要自己创建规则来加载helper功能， nf_conntrack_helper_try_module_get 会自动加载nf_conntrack_pptp这些模块吧
```text
iptables -A PREROUTING -t raw -p tcp --dport 21 -j CT --helper ftp
```

```c
static struct nf_conntrack_helper *
nf_ct_lookup_helper(struct nf_conn *ct, struct net *net)
{
	if (!net->ct.sysctl_auto_assign_helper) {
		if (net->ct.auto_assign_helper_warned)
			return NULL;
		if (!__nf_ct_helper_find(&ct->tuplehash[IP_CT_DIR_REPLY].tuple))
			return NULL;
		pr_info("nf_conntrack: default automatic helper assignment "
			"has been turned off for security reasons and CT-based "
			" firewall rule not found. Use the iptables CT target "
			"to attach helpers instead.\n");
		net->ct.auto_assign_helper_warned = 1;
		return NULL;
	}

	return __nf_ct_helper_find(&ct->tuplehash[IP_CT_DIR_REPLY].tuple);
}


int __nf_ct_try_assign_helper(struct nf_conn *ct, struct nf_conn *tmpl,
			      gfp_t flags)
{
	struct nf_conntrack_helper *helper = NULL;
	struct nf_conn_help *help;
	struct net *net = nf_ct_net(ct);

	/* We already got a helper explicitly attached. The function
	 * nf_conntrack_alter_reply - in case NAT is in use - asks for looking
	 * the helper up again. Since now the user is in full control of
	 * making consistent helper configurations, skip this automatic
	 * re-lookup, otherwise we'll lose the helper.
	 */
	if (test_bit(IPS_HELPER_BIT, &ct->status))
		return 0;

	if (tmpl != NULL) {
		help = nfct_help(tmpl);
		if (help != NULL) {
			helper = help->helper;
			set_bit(IPS_HELPER_BIT, &ct->status);
		}
	}

	help = nfct_help(ct);

	if (helper == NULL) {
		helper = nf_ct_lookup_helper(ct, net);
		if (helper == NULL) {
			if (help)
				RCU_INIT_POINTER(help->helper, NULL);
			return 0;
		}
	}

	if (help == NULL) {
		help = nf_ct_helper_ext_add(ct, helper, flags);
		if (help == NULL)
			return -ENOMEM;
	} else {
		/* We only allow helper re-assignment of the same sort since
		 * we cannot reallocate the helper extension area.
		 */
		struct nf_conntrack_helper *tmp = rcu_dereference(help->helper);

		if (tmp && tmp->help != helper->help) {
			RCU_INIT_POINTER(help->helper, NULL);
			return 0;
		}
	}

	rcu_assign_pointer(help->helper, helper);

	return 0;
}
EXPORT_SYMBOL_GPL(__nf_ct_try_assign_helper);



static int
xt_ct_set_helper(struct nf_conn *ct, const char *helper_name,
		 const struct xt_tgchk_param *par)
{
	struct nf_conntrack_helper *helper;
	struct nf_conn_help *help;
	u8 proto;

	proto = xt_ct_find_proto(par);
	if (!proto) {
		pr_info_ratelimited("You must specify a L4 protocol and not use inversions on it\n");
		return -ENOENT;
	}

	helper = nf_conntrack_helper_try_module_get(helper_name, par->family,
						    proto);
	if (helper == NULL) {
		pr_info_ratelimited("No such helper \"%s\"\n", helper_name);
		return -ENOENT;
	}

	help = nf_ct_helper_ext_add(ct, GFP_KERNEL);
	if (help == NULL) {
		nf_conntrack_helper_put(helper);
		return -ENOMEM;
	}

	rcu_assign_pointer(help->helper, helper);
	return 0;
}
```

         

  
