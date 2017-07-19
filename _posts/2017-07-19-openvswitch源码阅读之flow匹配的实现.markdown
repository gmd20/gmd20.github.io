[ovs datapath](http://docs.openvswitch.org/en/latest/topics/datapath/)
“UFID” 等概念的介绍
 

流表结构
=========
```text
struct datapath {
	struct rcu_head rcu;
	struct list_head list_node;

	/* Flow table. */
	struct flow_table table;     // 每个datapath有自己的流表，上来的包就是在这个流表里面搜索。但按照文档说法openflow有255个table的，不知道怎么从这个表跳到另外一个表。

	/* Switch ports. */
	struct hlist_head *ports;

	/* Stats. */
	struct dp_stats_percpu __percpu *stats_percpu;

	/* Network namespace ref. */
	possible_net_t net;

	u32 user_features;

	u32 max_headroom;
};



struct sw_flow {
	struct rcu_head rcu;
	struct {
		struct hlist_node node[2];
		u32 hash;
	} flow_table, ufid_table;
	int stats_last_writer;		/* CPU id of the last writer on
					 * 'stats[0]'.
					 */
	struct sw_flow_key key;         // 用来算hash查找匹配流的， 就是从网络包里面提取的以太网头部，tcp端口之类的信息，很大的一个结构
	struct sw_flow_id id;           // flow的UUID标识。底层有两个hash表，内核查找flow使用key来做哈希计算，用户空间用UFID计算哈希，在两个hash表里面走可以找到这个flow
	struct sw_flow_mask *mask;      // key的mask，类似ip地址的mask
	struct sw_flow_actions __rcu *sf_acts;
	struct flow_stats __rcu *stats[]; /* One for each CPU.  First one
					   * is allocated at flow creation time,
					   * the rest are allocated on demand
					   * while holding the 'stats[0].lock'.
					   */
};



struct table_instance {  // 第2步的哈希表的数据结构
	struct flex_array *buckets;
	unsigned int n_buckets;
	struct rcu_head rcu;
	int node_ver;
	u32 hash_seed;
	bool keep_flows;
};

struct flow_table {
	struct table_instance __rcu *ti;              // OVS_FLOW_ATTR_KEY 普通的哈希表
	struct table_instance __rcu *ufid_ti;         // OVS_FLOW_ATTR_UFID 如果flow有ufid就使用这个哈希表。 是一个UUID字符串，不知道怎么区别这两种
	struct mask_cache_entry __percpu *mask_cache;  // 每个cpu有个缓存mask cache，缓存这个flow使用的mask在mask_array里面的位置。是一个255个元素的哈希表
	struct mask_array __rcu *mask_array;      // 所有flow使用到的mask的一个列表
	unsigned long last_rehash;
	unsigned int count;
	unsigned int ufid_count;
};

ufid_ti 是供 “ovs-dpctl” 这些终端命令行通过指定flow的ufid(一个UUID）来操作flow用的？ 一个flow应该同时在两个hash表里面。如果是网络包的查找
就用key 算hash找到flow，如果是命令行就根据指定的UUID找到对应的flow吧。这UFID主要是为了方便用户使用吧


mask_cache缓存的东西
struct mask_cache_entry {
	u32 skb_hash;        //  linux内核的flow id
	u32 mask_index;      //  这个linux flow（连接）应该使用的ovs哪一条flow的，那个flow的mask在mask_array里面的index
};


网络包进来的处理和查找flow
======================
flow.c 和 flow_table.c
```text
ovs_vport_receive
  ovs_flow_key_extract
    key_extract       // 这里会把skb的头部的mac地址ip地址端口等信息复制出来放到flow结构的key里面
  ovs_dp_process_packet
    ovs_flow_tbl_lookup_stats
      flow_lookup
```


flow的查找算法
==============

```text
	/* Look up flow. */
	flow = ovs_flow_tbl_lookup_stats(&dp->table, key, skb_get_hash(skb),
					 &n_mask_hit);

skb_get_hash是linux内核提供的函数，应该是linux内核提供的flow标识，应该是根据ip
tcp 端口这些计算生成的，同一个tcp连接的所有skb应该算出来是同一个hash值，表示
是属于同一个流的网络包。后面ovs里面应该用这个做流标识，用了缓存flow对应的mask的index。
参见内核源码  http://elixir.free-electrons.com/linux/latest/source/net/core/flow_dissector.c
/**
 * __skb_get_hash: calculate a flow hash
 * @skb: sk_buff to calculate flow hash from
 *
 * This function calculates a flow hash based on src/dst addresses
 * and src/dst port numbers.  Sets hash in skb to non-zero hash value
 * on success, zero indicates no valid hash.  Also, sets l4_hash in skb
 * if hash is a canonical 4-tuple hash over transport ports.
 */
void __skb_get_hash(struct sk_buff *skb)
{
	struct flow_keys keys;
	u32 hash;

	__flow_hash_secret_init();

	hash = ___skb_get_hash(skb, &keys, hashrnd);

	__skb_set_sw_hash(skb, hash, flow_keys_have_l4(&keys));
}
EXPORT_SYMBOL(__skb_get_hash);



/*
 * mask_cache maps flow to probable mask. This cache is not tightly
 * coupled cache, It means updates to  mask list can result in inconsistent
 * cache entry in mask cache.
 * This is per cpu cache and is divided in MC_HASH_SEGS segments.
 * In case of a hash collision the entry is hashed in next segment.
 */
struct sw_flow *ovs_flow_tbl_lookup_stats(struct flow_table *tbl,
					  const struct sw_flow_key *key,
					  u32 skb_hash,
					  u32 *n_mask_hit)
{
	struct mask_array *ma = rcu_dereference(tbl->mask_array);
	struct table_instance *ti = rcu_dereference(tbl->ti);
	struct mask_cache_entry *entries, *ce;
	struct sw_flow *flow;
	u32 hash;
	int seg;

	*n_mask_hit = 0;
	if (unlikely(!skb_hash)) {
		u32 mask_index = 0;

		return flow_lookup(tbl, ti, ma, key, n_mask_hit, &mask_index);
	}

	/* Pre and post recirulation flows usually have the same skb_hash
	 * value. To avoid hash collisions, rehash the 'skb_hash' with
	 * 'recirc_id'.  */
	if (key->recirc_id)
		skb_hash = jhash_1word(skb_hash, key->recirc_id);

	ce = NULL;
	hash = skb_hash;    // 这个linux内核的flow标识，同一个流比如tcp连接应该有一致的hash值
	entries = this_cpu_ptr(tbl->mask_cache);    // 每个cpu核心都有per cpu cache

  // 根据前面定义的MC_HASH_ENTRIES常量，mask_cache是一个255个元素大小的哈希表，
  // 也就是说最多只能缓存255个flow，一般来说255也足够了吧。如果linux同时有255个
  // 以上的tcp连接不停频繁切换就会出现缓存不命中需要遍历做mask array的搜索了。
  // 这个255大小的“非开放链表式”hash表，对cpu缓存比较友好吧。而且看样子mask array
  // 数目应该是对应ovs里面配置的flow的条数的。如果一个table配置的flow条数不多，
  // 遍历一下也就是多做几次hash查找而已。
  //
  // MC_HASH_SEGS 应该是4，把skb_hash的4个字节的每个字节做一次查找，就是说在表
  // 里面探测了4个位置。
  /* Find the cache entry 'ce' to operate on. */
  for (seg = 0; seg < MC_HASH_SEGS; seg++) {
    int index = hash & (MC_HASH_ENTRIES - 1);
    struct mask_cache_entry *e;

		e = &entries[index];
		if (e->skb_hash == skb_hash) { // 找到这个linux flow应该使用哪个mask（rule）
			flow = flow_lookup(tbl, ti, ma, key, n_mask_hit,
					   &e->mask_index);
			if (!flow)
				e->skb_hash = 0;
			return flow;
		}

		if (!ce || e->skb_hash < ce->skb_hash)
			ce = e;  /* A better replacement cache candidate. */

		hash >>= MC_HASH_SHIFT;
	}

	/* Cache miss, do full lookup. */
	flow = flow_lookup(tbl, ti, ma, key, n_mask_hit, &ce->mask_index);
	if (flow)
		ce->skb_hash = skb_hash;

	return flow;
}


/* Flow lookup does full lookup on flow table. It starts with
 * mask from index passed in *index.
 */
static struct sw_flow *flow_lookup(struct flow_table *tbl,
				   struct table_instance *ti,
				   const struct mask_array *ma,
				   const struct sw_flow_key *key,
				   u32 *n_mask_hit,
				   u32 *index)
{
	struct sw_flow_mask *mask;
	struct sw_flow *flow;
	int i;

	if (*index < ma->max) {  // 快速路径 : 直接根据第1步cache命中的index找到mask，就可以直接到哈希表里面找flow了
		mask = rcu_dereference_ovsl(ma->masks[*index]);
		if (mask) {
			flow = masked_flow_lookup(ti, key, mask, n_mask_hit);     // 第2步在哈希表里面找flow
			if (flow)
				return flow;
		}
	}

  // 慢路径： 前面的cache没有命中，只能遍历mask_array数组所有mask，每一个都查找一遍看看有匹配的没有
	for (i = 0; i < ma->max; i++)  {

		if (i == *index)
			continue;

		mask = rcu_dereference_ovsl(ma->masks[i]);
		if (!mask)
			continue;
    
    // 注意这个对每个mask的规则，都用mask 对 key做掩码之后才计算hash值，然后
    // 采用这个hash值去表里面找。
		flow = masked_flow_lookup(ti, key, mask, n_mask_hit);  // 第2步在哈希表里面找flow
		if (flow) { /* Found */
			*index = i;     // 如果找到了匹配的flow，就把mask的index返回上次缓存起来。
			return flow;
		}
	}

	return NULL;
}


这个查找方法有点复杂，openvswith把它叫做"Megaflows"技术吧， 查找分为两步
第1步： 在 mask_array数组里面找到flow对应的mask，这里有一个简单的hash缓存的优化，可以避免顺序遍历数组搜索
第2步： 在 hash table里面找到flow，根据key + mask做完掩码之后才计算hash值来做哈希表查找。

ovs为什么分为两级的搜索，而不是每个linux flow 直接对应hash表里面的一项，那样只
做一次查找就可以了。听说一开始ovs是这么实现的，后来才改成先查找mask的方式，最近
才又引入了mask的hash查找加速。想了一下，如果每个linux的flow都建一个hash项的话，
确实数目可能比较多，导致hash表比较大，消耗内存多性能也不见的好。它搞成这种mask
归类匹配的话，也是和上层配置流的通配符wilcard支持相匹配的，归类之后需要创建的flow
的hash项就少很多了。  比如  “192.168.1.0/24 port=*” 一条flow就能匹配多个192.168.1.1
192.168.1.2 等多个ip的tcp连接了。如果直接使用linux flow对应的方式就需要255 * 65535
条hash项。肯定还是基于这种mask通配符的匹配更好了，基本ovs里面配置多少条flow就
只需要创建多少条 flow+mask的匹配项而已。



往表中插入flow比较容易理解
/* Must be called with OVS mutex held. */
int ovs_flow_tbl_insert(struct flow_table *table, struct sw_flow *flow,
			const struct sw_flow_mask *mask)
{
	int err;

	err = flow_mask_insert(table, flow, mask);   //  添加mask
	if (err)
		return err;
	flow_key_insert(table, flow);            // 添加key对应hash项
	if (ovs_identifier_is_ufid(&flow->id))
		flow_ufid_insert(table, flow);        // 添加ufid对应的hash项

	return 0;
}
```


ovs的resubmit这个action可以把包提交到另外table进行匹配时怎么实现的
==================================================================
```text
ovs应该只是openflow标准，可以有255个table，应该可以在多个table串联
出复杂的pipeline处理流程。

https://networkheresy.com/2014/11/13/accelerating-open-vswitch-to-ludicrous-speed/
这个megaflow技术确实大大提高了ovs的性能。
看这里的说法openvswitch的flow的创建是在用户空间的vswitchd里面创建的，我之前
一篇文章也概读了一下那个xlate的代码了。但table的跳转，还没有看到代码是怎么实现的。
感觉会不会是xlate在生成flow的action的时候，已经把所有的255个table合并在一起了。
上层推算的时候就完全可以把table 的区分去掉了。

xlate_ofpact_resubmit 
  xlate_table_action
    xlate_resubmit_resource_check

看看这几个函数代码，好像有点像是跟猜测的那样实现的，底层内核确实可以做到不区分table_id了
```
