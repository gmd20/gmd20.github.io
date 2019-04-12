创建的新的interface的ifindex是不停的增长的，可以通过 ip link 来查看
```c
/**
 *	dev_new_index	-	allocate an ifindex
 *	@net: the applicable net namespace
 *
 *	Returns a suitable unique value for a new device interface
 *	number.  The caller must hold the rtnl semaphore or the
 *	dev_base_lock to be sure it remains unique.
 */
static int dev_new_index(struct net *net)
{
	int ifindex = net->ifindex;

	for (;;) {
		if (++ifindex <= 0)
			ifindex = 1;
		if (!__dev_get_by_index(net, ifindex))
			return net->ifindex = ifindex;
	}
}
```

不过手工创建时应该是可以指定index的
```text
# ip link add name bond0 type bond
# /usr/sbin/ip link add name bond0 index 10 type bond  
# ip link delete bond0
```
