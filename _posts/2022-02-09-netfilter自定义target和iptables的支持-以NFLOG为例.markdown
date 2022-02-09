http://git.netfilter.org/iptables/tree/extensions/libipt_ULOG.c    

https://elixir.bootlin.com/linux/latest/source/net/netfilter/nfnetlink_log.c     
https://elixir.bootlin.com/linux/latest/source/net/netfilter/nf_log_syslog.c   
https://elixir.bootlin.com/linux/latest/source/net/netfilter/nf_log.c    
https://elixir.bootlin.com/linux/latest/source/net/netfilter/xt_NFLOG.c    
https://elixir.bootlin.com/linux/latest/source/net/netfilter/xt_LOG.c    


```text

# modinfo ipt_LOG
filename:       /lib/modules/4.18.0-348.7.1.el8_5.x86_64/kernel/net/netfilter/xt_LOG.ko.xz

# modinfo xt_NFLOG
filename:       /lib/modules/4.18.0-348.12.2.el8_5.x86_64/kernel/net/netfilter/xt_NFLOG.ko.xz
softdep:        pre: nfnetlink_log
alias:          ip6t_NFLOG
alias:          ipt_NFLOG
license:        GPL



strace iptables -t filter -A INPUT -s 119.29.29.29 -j GMD20
回去自动加载/usr/lib64/xtables/libxt_GMD20.so应用层动态链接库和相关的内核文件

https://elixir.bootlin.com/linux/latest/source/net/netfilter/x_tables.c
xt_request_find_target
xt_target_to_user
copy_entries_to_user
nft_target_select_ops

http://git.netfilter.org/iptables/tree/libxtables/xtables.c
/proc/sys/kernel/modprobe
xtables_find_target
  compatible_target_revision
    xtables_compatible_revision()
      xtables_load_ko()


/lib64/libxtables.so.12
```

整个框架还是比较方便实现自定义的 target之类的扩展的



内核里面LOG是写直接写syslog日志，NFLOG是netlink把网络包都发送到用户空间再写日志的吧
···text
void nf_log_packet(struct net *net,
		   u_int8_t pf,
		   unsigned int hooknum,
		   const struct sk_buff *skb,
		   const struct net_device *in,
		   const struct net_device *out,
		   const struct nf_loginfo *loginfo,
		   const char *fmt, ...)
{
	va_list args;
	char prefix[NF_LOG_PREFIXLEN];
	const struct nf_logger *logger;

	rcu_read_lock();
	if (loginfo != NULL)
		logger = rcu_dereference(loggers[pf][loginfo->type]);
	else
		logger = rcu_dereference(net->nf.nf_loggers[pf]);

	if (logger) {
		va_start(args, fmt);
		vsnprintf(prefix, sizeof(prefix), fmt, args);
		va_end(args);
		logger->logfn(net, pf, hooknum, skb, in, out, loginfo, prefix);
	}
	rcu_read_unlock();
}
EXPORT_SYMBOL(nf_log_packet);

static void nf_log_ip_packet(struct net *net, u_int8_t pf,
			     unsigned int hooknum, const struct sk_buff *skb,
			     const struct net_device *in,
			     const struct net_device *out,
			     const struct nf_loginfo *loginfo,
			     const char *prefix)
{
	struct nf_log_buf *m;

	/* FIXME: Disabled from containers until syslog ns is supported */
	if (!net_eq(net, &init_net) && !sysctl_nf_log_all_netns)
		return;

	m = nf_log_buf_open();

	if (!loginfo)
		loginfo = &default_loginfo;

	nf_log_dump_packet_common(m, pf, hooknum, skb, in,
				  out, loginfo, prefix);

	if (in)
		dump_ipv4_mac_header(m, loginfo, skb);

	dump_ipv4_packet(net, m, loginfo, skb, 0);

	nf_log_buf_close(m);
}

static struct nf_logger nf_ip_logger __read_mostly = {
	.name		= "nf_log_ipv4",
	.type		= NF_LOG_TYPE_LOG,
	.logfn		= nf_log_ip_packet,
	.me		= THIS_MODULE,
};
```
