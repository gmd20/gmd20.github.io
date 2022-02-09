http://git.netfilter.org/iptables/tree/extensions/libipt_ULOG.c
https://elixir.bootlin.com/linux/latest/source/net/netfilter/nfnetlink_log.c#L855
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
