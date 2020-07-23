https://elixir.bootlin.com/linux/latest/source/include/net/netfilter/nf_conntrack_zones.h
https://lwn.net/Articles/370152/
https://specs.openstack.org/openstack/neutron-specs/specs/liberty/conntrack-zones.html

才发现conntrack有一个zone的东西， 可以查看代码里面nf_ct_zone 等函数的实现，应该是用来隔离网络的，查找时的key包含zone id进去了吧。
这样应该两个 不同网络设备下用一样  IP地址，因为zone id可以配置的不一样的，最后的 conntrack也不会有冲突吧。

好像有些虚拟网络可以用来隔离不同的租户的？？
