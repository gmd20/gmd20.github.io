sk_buff 有几个字段是标识了这条网络连接（流）的hash。
```text
 *	@hash: the packet hash
 *	@l4_hash: indicate hash is a canonical 4-tuple hash over transport
 *		ports.
 *	@sw_hash: indicates hash was computed in software stack
```
这里通常是网卡硬件在硬件里面计算出来的了。
使用ethtool命令可以查看和设置 网卡用哪些字段来计算哈希。

```text
查看网卡里面流的hash计算来源
ethtool -n eth1 rx-flow-hash tcp4
TCP over IPV4 flows use these fields for computing Hash flow key:
IP SA
IP DA
L4 bytes 0 & 1 [TCP/UDP src port]
L4 bytes 2 & 3 [TCP/UDP dst port]

设置udp流的hash使用 源ip和目的ip和源端口目的端口计算拿出来
ethtool -N eth1 rx-flow-hash udp4 sdfn

详细参考ethtool的用法
https://www.man7.org/linux/man-pages/man8/ethtool.8.html
```
但这个东西应该跟硬件有关系，在Intel的igb网卡上设置 “ethtool -N eth1 rx-flow-hash udp4 sd” 是成功的，但  “ethtool -N eth1 rx-flow-hash tcp4 sd”  不成功，
看驱动代码的话可以看到拒绝这么设置了。 就是允许udp流只用 双向ip不用端口，但tcp流没法把双向的端口排除出hash值。
Intel 网卡的datasheet提到rxhash只使用源ip和目的ip来计算hash的函数设置，可能有些硬件是支持这么配置的，高级的网卡还支持配置 流的过滤规则。
但可能驱动为了兼容多中网卡类似，分的不是很细，主流的网卡是调整不了这个hash的来源了。 完全禁止网卡的rxhash倒是可以的。
“ethtool   -K rxhash off eth0” 吧
l4_hash的话就是hash里面包含了tcp udp的端口的吧。

igb 驱动说自己的hash不是 4层的l4_hash。 但ethtool -n eth1 rx-flow-hash tcp4 有说包含端口的，有点不一致。
```c
static inline void igb_rx_hash(struct igb_ring *ring,
			       union e1000_adv_rx_desc *rx_desc,
			       struct sk_buff *skb)
{
	if (ring->netdev->features & NETIF_F_RXHASH)
		skb_set_hash(skb,
			     le32_to_cpu(rx_desc->wb.lower.hi_dword.rss),
			     PKT_HASH_TYPE_L3);
}
```

如果硬件不支持 “流hash”，只能软件解析包头的ip和端口来计算了 代码在 flow_dissector.c里面，这个是网卡的RSS RFS负载均衡分流到不同cpu时用到吧。
主要目的是优化cpu cache效率，同一个流分到同一个cpu上应用进行处理效果更好吧。但别的地方比如 qos里面做限速时有的也可以根据这个hash分流到不同的
丢列里面去出来。
```c
static inline __u32 skb_get_hash_raw(const struct sk_buff *skb)
{
	return skb->hash;
}

static inline __u32 skb_get_hash(struct sk_buff *skb)
{
	if (!skb->l4_hash && !skb->sw_hash)
		__skb_get_hash(skb);

	return skb->hash;
}

static inline u32 __flow_hash_from_keys(struct flow_keys *keys,
					const siphash_key_t *keyval)
{
	u32 hash;

	__flow_hash_consistentify(keys);

	hash = siphash(flow_keys_hash_start(keys),
		       flow_keys_hash_length(keys), keyval);
	if (!hash)
		hash = 1;

	return hash;
}

u32 flow_hash_from_keys(struct flow_keys *keys)
{
	__flow_hash_secret_init();
	return __flow_hash_from_keys(keys, &hashrnd);
}
EXPORT_SYMBOL(flow_hash_from_keys);


static inline u32 ___skb_get_hash(const struct sk_buff *skb,
				  struct flow_keys *keys,
				  const siphash_key_t *keyval)
{
  // __skb_flow_dissect 解析各种协议的ip地址和端口出来存到key里面;
	skb_flow_dissect_flow_keys(skb, keys,
				   FLOW_DISSECTOR_F_STOP_AT_FLOW_LABEL);

	return __flow_hash_from_keys(keys, keyval);
}

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

	hash = ___skb_get_hash(skb, &keys, &hashrnd);

	__skb_set_sw_hash(skb, hash, flow_keys_have_l4(&keys));
}
EXPORT_SYMBOL(__skb_get_hash)
```

RFS: Receive Flow Steering 相关的文档
https://www.kernel.org/doc/html/latest/networking/scaling.html


