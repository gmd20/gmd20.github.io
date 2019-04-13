# 可以通过 ip link 来查看各个接口的 ifindex

# 写代码获取ifindex可以用if_nametoindex, if_indextoname 这两个函数吧
http://man7.org/linux/man-pages/man3/if_nameindex.3.html

libnl可以用来辅助创建virtual link
```c
int rtnl_link_add	(	struct nl_sock * 	sk,
struct rtnl_link * 	link,
int 	flags 
)		
Add virtual link.

Parameters
sk	netlink socket.
link	new link to add
flags	additional netlink message flags
Builds a RTM_NEWLINK netlink message requesting the addition of a new virtual link.

After sending, the function will wait for the ACK or an eventual error message to be received and will therefore block until the operation has been completed.

Definition at line 1404 of file link.c.

References nl_send_sync(), and rtnl_link_build_add_request().

Referenced by rtnl_link_bond_add(), rtnl_link_bridge_add(), and rtnl_link_veth_add().

设置ifindex
extern void	rtnl_link_set_ifindex(struct rtnl_link *, int);
extern int	rtnl_link_get_ifindex(struct rtnl_link *);
```

# 创建的新的interface（net_device), 如果不指定内核创建时默认的ifindex是不停的增长的，
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

# 不过手工创建时应该是可以指定ifindex的，只要没被别的接口占用就可以
```text
# ip link add name bond0 type bond
# /usr/sbin/ip link add name bond0 index 10 type bond  
# ip link delete bond0
```

# 代码里面调用rtnelink接口来创建是应该也是可以指定 if_index的
参考 http://man7.org/linux/man-pages/man7/rtnetlink.7.html
```text
      RTM_NEWLINK, RTM_DELLINK, RTM_GETLINK
              Create, remove or get information about a specific network
              interface.  These messages contain an ifinfomsg structure fol‐
              lowed by a series of rtattr structures.

              struct ifinfomsg {
                  unsigned char  ifi_family; /* AF_UNSPEC */
                  unsigned short ifi_type;   /* Device type */
                  int            ifi_index;  /* Interface index */
                  unsigned int   ifi_flags;  /* Device flags  */
                  unsigned int   ifi_change; /* change mask */
              };

              ifi_flags contains the device flags, see netdevice(7);
              ifi_index is the unique interface index (since Linux 3.7, it
              is possible to feed a nonzero value with the RTM_NEWLINK mes‐
              sage, thus creating a link with the given ifindex); ifi_change
              is reserved for future use and should be always set to
              0xFFFFFFFF.
```
