内核有一个函数inet_lookup_ifaddr_rcu，但这个函数没有导出的，也可以inet_addr_type_dev_table这个获取路由项是否为本地ip来判断。
ip_dev_find应该可以可以通过查找ip在哪个设备上来确人，不过这个麻烦些完了需要dev_put(dev);释放掉资源


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



static inline struct net_device *ip_dev_find(struct net *net, __be32 addr)
{
	return __ip_dev_find(net, addr, true);
}

/**
 * __ip_dev_find - find the first device with a given source address.
 * @net: the net namespace
 * @addr: the source address
 * @devref: if true, take a reference on the found device
 *
 * If a caller uses devref=false, it should be protected by RCU, or RTNL
 */
struct net_device *__ip_dev_find(struct net *net, __be32 addr, bool devref)
{
	struct net_device *result = NULL;
	struct in_ifaddr *ifa;

	rcu_read_lock();
	ifa = inet_lookup_ifaddr_rcu(net, addr);
	if (!ifa) {
		struct flowi4 fl4 = { .daddr = addr };
		struct fib_result res = { 0 };
		struct fib_table *local;

		/* Fallback to FIB local table so that communication
		 * over loopback subnets work.
		 */
		local = fib_get_table(net, RT_TABLE_LOCAL);
		if (local &&
		    !fib_table_lookup(local, &fl4, &res, FIB_LOOKUP_NOREF) &&
		    res.type == RTN_LOCAL)
			result = FIB_RES_DEV(res);
	} else {
		result = ifa->ifa_dev->dev;
	}
	if (result && devref)
		dev_hold(result);
	rcu_read_unlock();
	return result;
}
EXPORT_SYMBOL(__ip_dev_find);


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
