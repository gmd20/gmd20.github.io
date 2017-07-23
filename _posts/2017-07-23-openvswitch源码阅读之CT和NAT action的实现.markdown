用户空间解析conntrack/NAT action
================================

```text
ofproto/ofproto-dpif-xlate,c
  compose_conntrack_action()
     OVS_ACTION_ATTR_CT    // 把CT的action配置信息放到一定的结构里传递到内核空间
```

内核收到用户空间发过来的action
==============================
```text
flow_netlink.c
  __ovs_nla_copy_actions
    datapath/conntrack.c
      ovs_ct_copy_action
        parse_ct
        		case OVS_CT_ATTR_NAT: {
              int err = parse_nat(a, info, log);   // NAT属于CT的子项

解析之后的信息应该是存在这个结构里面吧

/* Conntrack action context for execution. */
struct ovs_conntrack_info {
	struct nf_conntrack_helper *helper;
	struct nf_conntrack_zone zone;
	struct nf_conn *ct;
	u8 commit : 1;
	u8 nat : 3;                 /* enum ovs_ct_nat */
	u8 random_fully_compat : 1; /* bool */
	u8 force : 1;
	u8 have_eventmask : 1;
	u16 family;
	u32 eventmask;              /* Mask of 1 << IPCT_*. */
	struct md_mark mark;
	struct md_labels labels;
#ifdef CONFIG_NF_NAT_NEEDED
	struct nf_nat_range range;  /* Only present for SRC NAT and DST NAT. */
#endif
};


最后把上面的内容保存到 struct sw_flow_actions 里面去
	err = ovs_nla_add_action(sfa, OVS_ACTION_ATTR_CT, &ct_info,
				 sizeof(ct_info), log);

```

conntrack/NATaction的执行
=========================
```text

datapath/actions.c
  /* Execute a list of actions against 'skb'. */
  static int do_execute_actions(struct datapath *dp, struct sk_buff *skb,
    case OVS_ACTION_ATTR_CT:
        err = ovs_ct_execute(ovs_dp_get_net(dp), skb, key,
                 nla_data(a));


datapath/conntrack.c
  int ovs_ct_execute(struct net *net, struct sk_buff *skb,
         struct sw_flow_key *key,
         const struct ovs_conntrack_info *info)
      ovs_ct_commit
        __ovs_ct_lookup
          nf_conntrack_in
            ovs_ct_update_key
          nf_ct_get        // nf开头的都是linux内核函数了
          ovs_ct_nat（）     // 这里是真正的执行吧
        ovs_ct_set_mark      // ovs 的 action 相关的属性
        ovs_ct_set_labels


/* Returns NF_DROP if the packet should be dropped, NF_ACCEPT otherwise. */
static int ovs_ct_nat(struct net *net, struct sw_flow_key *key,
		      const struct ovs_conntrack_info *info,
		      struct sk_buff *skb, struct nf_conn *ct,
		      enum ip_conntrack_info ctinfo)
     ovs_ct_nat_execute(skb, ct, ctinfo, &info->range, maniptype);


static int ovs_ct_nat_execute(struct sk_buff *skb, struct nf_conn *ct,
			      enum ip_conntrack_info ctinfo,
			      const struct nf_nat_range *range,
			      enum nf_nat_manip_type maniptype)
   err = nf_nat_packet(ct, ctinfo, hooknum, skb);  // 实际的工作还是在linux内核的函数负责的


http://elixir.free-electrons.com/linux/latest/source/net/netfilter/nf_nat_core.c
/* Do packet manipulations according to nf_nat_setup_info. */
unsigned int nf_nat_packet(struct nf_conn *ct,
			   enum ip_conntrack_info ctinfo,
			   unsigned int hooknum,
			   struct sk_buff *skb)
     if (!l3proto->manip_pkt(skb, 0, l4proto, &target, mtype))

```


总结一下
========

从这个看openvswithc的 CT/NAT 还是使用linux的内核那一套。跟iptables的NAT效果应该
是一样的，ovs的文档也是这么说的。 就是 ovs的DPDK的datapath的时候，全部是用户
态执行，应该用不了linux内核这个代码了， 但intel的ovs dpdk文档还说也支持CT的，
估计那个要另外实现一套了。但DPDK应该还不支持NAT吧？


