```text

static const struct nf_hook_ops ipv4_conntrack_ops[] = {
	{
		.hook		= ipv4_conntrack_in,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK,
	},


ip_rcv 所有进入系统的包
  NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING
    ipv4_conntrack_in
      nf_conntrack_in
        resolve_normal_ct
          __nf_conntrack_find_get  // 先在表里面查找
          init_conntrack  // 如果找不到就创建 和初始化连接ct结构，
          nf_ct_set(skb, ct, ctinfo);  // 设置 skb的 nfct 结构


可以使用skb_nfct 或者 nf_ct_get 函数来获取skb的 连接信息。
```
