对应的源码在这几个目录下
------------------------
```text
linux-4.11\net\netfilter\nf_conntrack*
linux-4.11\net\ipv4\netfilter\nf_conntrack*
linux-4.11\net\ipv6\netfilter\nf_conntrack*
```

netfiler的挂载点
----------------

```c
/* Connection tracking may drop packets, but never alters them, so
   make it the first hook. */
static struct nf_hook_ops ipv4_conntrack_ops[] __read_mostly = {
	{
		.hook		= ipv4_conntrack_in,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK,
	},
	{
		.hook		= ipv4_conntrack_local,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_CONNTRACK,
	},
	{
		.hook		= ipv4_helper,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK_HELPER,
	},
	{
		.hook		= ipv4_confirm,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM,
	},
	{
		.hook		= ipv4_helper,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_CONNTRACK_HELPER,
	},
	{
		.hook		= ipv4_confirm,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM,
	},
};
```

ipv4_hooks_register  里面注册上面的netfiler hook点


下面这个是ipv4的 conntrack的接口
--------------------------------
tuple应该就是说要里面网络包里面的那些字节来做 connection tracking状态。对应ip包，肯定就是包头里面的ip地址了。
```c
struct nf_conntrack_l3proto nf_conntrack_l3proto_ipv4 __read_mostly = {
	.l3proto	 = PF_INET,
	.name		 = "ipv4",
	.pkt_to_tuple	 = ipv4_pkt_to_tuple,
	.invert_tuple	 = ipv4_invert_tuple,
	.print_tuple	 = ipv4_print_tuple,
	.get_l4proto	 = ipv4_get_l4proto,
#if IS_ENABLED(CONFIG_NF_CT_NETLINK)
	.tuple_to_nlattr = ipv4_tuple_to_nlattr,
	.nlattr_tuple_size = ipv4_nlattr_tuple_size,
	.nlattr_to_tuple = ipv4_nlattr_to_tuple,
	.nla_policy	 = ipv4_nla_policy,
#endif
	.net_ns_get	 = ipv4_hooks_register,
	.net_ns_put	 = ipv4_hooks_unregister,
	.me		 = THIS_MODULE,
};
```

对进来的网络包的处理流程
------------------------
```text
ipv4_conntrack_in/ipv6_conntrack_in
   nf_conntrack_in   // 对进来的网络包，hook函数都会调用这个函数，在这里面进行connection tracking 状态的维护吧
      resolve_normal_ct
         hash_conntrack_raw
         __nf_conntrack_find_get
           nf_ct_set(skb, ct, *ctinfo);   // ct结构创建之后会赋值给 skb->_nfct ，让两者关联起来
            
```

根据tuple字节查找连接的ct结构
-----------------------------			
```text
/* Find a connection corresponding to a tuple. */
nf_conntrack_find_get
    ____nf_conntrack_find

```

哈希函数
--------
```text
hash_conntrack_raw
   jhash2


/* jhash2 - hash an array of u32's
 * @k: the key which must be an array of u32's
 * @length: the number of u32's in the key
 * @initval: the previous hash, or an arbitray value
 *
 * Returns the hash value of the key.
 */
static inline u32 jhash2(const u32 *k, u32 length, u32 initval)

/* jhash.h: Jenkins hash support.
 *
 * Copyright (C) 2006. Bob Jenkins (bob_jenkins@burtleburtle.net)
 *
 * http://burtleburtle.net/bob/hash/
 *
 * These are the credits from Bob's sources:
 *
 * lookup3.c, by Bob Jenkins, May 2006, Public Domain.
```
很出名的“Jenkins hash ”， 速度很快，在Facebook开源的folly c++ 库里面也有实现。







看看这个哈希表实现
------------------


* 1. nf_conntrack_get_ht(&ct_hash, &hsize);
使用了一个“ticket号码加重试”无锁机制来获取一致的哈希表指针和哈希表大小  &ct_hash, &hsize。
现在才知道内核也用了这种新潮的技术了。 学了一招，也有内核代码里面有合适的场合也可以试用一下

latest/source/include/linux/seqlock.h
```
/*
 * Reader/writer consistent mechanism without starving writers. This type of
 * lock for data where the reader wants a consistent set of information
 * and is willing to retry if the information changes. There are two types
 * of readers:
 * 1. Sequence readers which never block a writer but they may have to retry
 *    if a writer is in progress by detecting change in sequence number.
 *    Writers do not wait for a sequence reader.
 * 2. Locking readers which will wait if a writer or another locking reader
 *    is in progress. A locking reader in progress will also block a writer
 *    from going forward. Unlike the regular rwlock, the read lock here is
 *    exclusive so that only one locking reader can get it.
 *
 * This is not as cache friendly as brlock. Also, this may not work well
 * for data that contains pointers, because any writer could
 * invalidate a pointer that a reader was following.
 *
 * Expected non-blocking reader usage:
 * 	do {
 *	    seq = read_seqbegin(&foo);
 * 	...
 *      } while (read_seqretry(&foo, seq));
 *
 *
 * On non-SMP the spin locks disappear but the writer still needs
 * to increment the sequence variables because an interrupt routine could
 * change the state of the data.
 *
 * Based on x86_64 vsyscall gettimeofday 
 * by Keith Owens and Andrea Arcangeli
 */

```

*2.bucket = reciprocal_scale(hash, hsize);
```
/**
 * reciprocal_scale - "scale" a value into range [0, ep_ro)
 * @val: value
 * @ep_ro: right open interval endpoint
 *
 * Perform a "reciprocal multiplication" in order to "scale" a value into
 * range [0, ep_ro), where the upper interval endpoint is right-open.
 * This is useful, e.g. for accessing a index of an array containing
 * ep_ro elements, for example. Think of it as sort of modulus, only that
 * the result isn't that of modulo. ;) Note that if initial input is a
 * small value, then result will return 0.
 *
 * Return: a result based on val in interval [0, ep_ro).
 */
static inline u32 reciprocal_scale(u32 val, u32 ep_ro)
{
	return (u32)(((u64) val * ep_ro) >> 32);
}
```
这个哈希表计算bucket的index的时候 不是通常的 hash_value % hash_size
或者  hash_value & hash_size_mask 这样的计算，而是通过下面这种乘除法来把32位的数值范围
缩小到 0到hash_size范围来。
```text

               (32bit hash_value) * hash_size
bucket_index = ------------------------------
                          2^32
```

第一次见到用这种方法的，可能 mask的方法要求size必须为2的指数，这个方法对size的设置可以更灵活一些吧。



* 3. hlist_nulls_for_each_entry_rcu(h, n, &ct_hash[bucket], hnnode) {

```text
	hlist_nulls_for_each_entry_rcu(h, n, &ct_hash[bucket], hnnode) {
		struct nf_conn *ct;

		ct = nf_ct_tuplehash_to_ctrack(h);
		if (nf_ct_is_expired(ct)) {
			nf_ct_gc_expired(ct);
			continue;
		}

		if (nf_ct_is_dying(ct))
			continue;

		if (nf_ct_key_equal(h, tuple, zone, net))   // 二进制比较 tuple字节
			return h;
	}
```
这个bucket的链表使用了RCU技术的，看来优化什么的做的很到位啊！！！



* 4. 能够在这个struct nf_conn *ct; 里面保存一下自定义的东西呢？
http://elixir.free-electrons.com/linux/latest/source/include/net/netfilter/nf_conntrack.h#L75
这样对每个数据包 经过conntack后，都可以通过 skb->_nfct 拿到struct nf_conn 进而拿到自定义的结构就完美了
```c
struct nf_conn {
	/* Usage count in here is 1 for hash table, 1 per skb,
	 * plus 1 for any connection(s) we are `master' for
	 *
	 * Hint, SKB address this struct and refcnt via skb->_nfct and
	 * helpers nf_conntrack_get() and nf_conntrack_put().
	 * Helper nf_ct_put() equals nf_conntrack_put() by dec refcnt,
	 * beware nf_ct_get() is different and don't inc refcnt.
	 */

}
  
static inline void
nf_ct_set(struct sk_buff *skb, struct nf_conn *ct, enum ip_conntrack_info info)
{
	skb->_nfct = (unsigned long)ct | info;
}

/* Return conntrack_info and tuple hash for given skb. */
static inline struct nf_conn *
nf_ct_get(const struct sk_buff *skb, enum ip_conntrack_info *ctinfo)
{
	*ctinfo = skb->_nfct & NFCT_INFOMASK;

	return (struct nf_conn *)(skb->_nfct & NFCT_PTRMASK);
}

```



网上有人试了一下
http://blog.csdn.net/dog250/article/details/23001113
可以把东西保存到这个机构，然后输出到
/proc/net/nf_conntrack
但需要修改linux内核文件



总结：
-----
这个conntrack的实现还是不错的，openvswitch里面也支持了，看linux版本的（linux-4.11\net\openvswitch ）应该也是用的内核这套代码吧。
 



