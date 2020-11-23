内核有一个函数inet_lookup_ifaddr_rcu，但这个函数没有导出的，也可以inet_addr_type_dev_table这个获取路由项是否为本地ip来判断。


```c
/* called under RCU lock */
struct in_ifaddr *inet_lookup_ifaddr_rcu(struct net *net, __be32 addr)
{
	u32 hash = inet_addr_hash(net, addr);
	struct in_ifaddr *ifa;

	hlist_for_each_entry_rcu(ifa, &inet_addr_lst[hash], hash)
		if (ifa->ifa_local == addr &&
		    net_eq(dev_net(ifa->ifa_dev->dev), net))
			return ifa;

	return NULL;
}

inet_addr_type_dev_table(net, dev, tip) == RTN_LOCAL
```

最笨的版本，遍历所有网卡上的ip地址
```c
	rcu_read_lock();
	for_each_netdev_rcu(&init_net, dev) {
		in_dev = __in_dev_get_rcu(dev);
		if (in_dev == NULL) {
			continue;
		}
		in_dev_for_each_ifa_rcu(ifa, in_dev) {
		}
	}
	rcu_read_unlock();

```
