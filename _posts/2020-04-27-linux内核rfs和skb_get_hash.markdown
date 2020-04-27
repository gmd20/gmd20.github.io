https://www.kernel.org/doc/Documentation/networking/scaling.txt

# 驱动层会设置skb hash，这个在很多地方都会用到，比如fq_codel 算法里面
https://elixir.bootlin.com/linux/latest/source/drivers/net/ethernet/intel/igb/igb_main.c#L8184
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

# socket层会设置flow 要放在那个的cpu处理？

https://elixir.bootlin.com/linux/latest/source/include/net/sock.h#L986
```c
static inline void sock_rps_record_flow(const struct sock *sk)
{
#ifdef CONFIG_RPS
	if (static_branch_unlikely(&rfs_needed)) {
		/* Reading sk->sk_rxhash might incur an expensive cache line
		 * miss.
		 *
		 * TCP_ESTABLISHED does cover almost all states where RFS
		 * might be useful, and is cheaper [1] than testing :
		 *	IPv4: inet_sk(sk)->inet_daddr
		 * 	IPv6: ipv6_addr_any(&sk->sk_v6_daddr)
		 * OR	an additional socket flag
		 * [1] : sk_state and sk_prot are in the same cache line.
		 */
		if (sk->sk_state == TCP_ESTABLISHED)
			sock_rps_record_flow_hash(sk->sk_rxhash);
	}
#endif
}

static inline void sock_rps_record_flow_hash(__u32 hash)
{
#ifdef CONFIG_RPS
	struct rps_sock_flow_table *sock_flow_table;

	rcu_read_lock();
	sock_flow_table = rcu_dereference(rps_sock_flow_table);
	rps_record_sock_flow(sock_flow_table, hash);
	rcu_read_unlock();
#endif
}

static inline void sock_rps_save_rxhash(struct sock *sk,
					const struct sk_buff *skb)
{
#ifdef CONFIG_RPS
	if (unlikely(sk->sk_rxhash != skb->hash))
		sk->sk_rxhash = skb->hash;
#endif
}


```
https://elixir.bootlin.com/linux/latest/source/include/linux/netdevice.h#L716
```c
static inline void rps_record_sock_flow(struct rps_sock_flow_table *table,
					u32 hash)
{
	if (table && hash) {
		unsigned int index = hash & table->mask;
		u32 val = hash & ~rps_cpu_mask;

		/* We only give a hint, preemption can change CPU under us */
		val |= raw_smp_processor_id();

		if (table->ents[index] != val)
			table->ents[index] = val;
	}
}
```

# 网卡中断会判断包发往哪个cpu进行处理
https://elixir.bootlin.com/linux/latest/source/net/core/dev.c#L4181
```c
/*
 * get_rps_cpu is called from netif_receive_skb and returns the target
 * CPU from the RPS map of the receiving queue for a given skb.
 * rcu_read_lock must be held on entry.
 */
static int get_rps_cpu(struct net_device *dev, struct sk_buff *skb,
		       struct rps_dev_flow **rflowp)
{
	const struct rps_sock_flow_table *sock_flow_table;
	struct netdev_rx_queue *rxqueue = dev->_rx;
	struct rps_dev_flow_table *flow_table;
	struct rps_map *map;
	int cpu = -1;
	u32 tcpu;
	u32 hash;

	if (skb_rx_queue_recorded(skb)) {
		u16 index = skb_get_rx_queue(skb);

		if (unlikely(index >= dev->real_num_rx_queues)) {
			WARN_ONCE(dev->real_num_rx_queues > 1,
				  "%s received packet on queue %u, but number "
				  "of RX queues is %u\n",
				  dev->name, index, dev->real_num_rx_queues);
			goto done;
		}
		rxqueue += index;
	}

	/* Avoid computing hash if RFS/RPS is not active for this rxqueue */

	flow_table = rcu_dereference(rxqueue->rps_flow_table);
	map = rcu_dereference(rxqueue->rps_map);
	if (!flow_table && !map)
		goto done;

	skb_reset_network_header(skb);
	hash = skb_get_hash(skb);
	if (!hash)
		goto done;

	sock_flow_table = rcu_dereference(rps_sock_flow_table);
	if (flow_table && sock_flow_table) {
		struct rps_dev_flow *rflow;
		u32 next_cpu;
		u32 ident;

		/* First check into global flow table if there is a match */
		ident = sock_flow_table->ents[hash & sock_flow_table->mask];
		if ((ident ^ hash) & ~rps_cpu_mask)
			goto try_rps;

		next_cpu = ident & rps_cpu_mask;

		/* OK, now we know there is a match,
		 * we can look at the local (per receive queue) flow table
		 */
		rflow = &flow_table->flows[hash & flow_table->mask];
		tcpu = rflow->cpu;

		/*
		 * If the desired CPU (where last recvmsg was done) is
		 * different from current CPU (one in the rx-queue flow
		 * table entry), switch if one of the following holds:
		 *   - Current CPU is unset (>= nr_cpu_ids).
		 *   - Current CPU is offline.
		 *   - The current CPU's queue tail has advanced beyond the
		 *     last packet that was enqueued using this table entry.
		 *     This guarantees that all previous packets for the flow
		 *     have been dequeued, thus preserving in order delivery.
		 */
		if (unlikely(tcpu != next_cpu) &&
		    (tcpu >= nr_cpu_ids || !cpu_online(tcpu) ||
		     ((int)(per_cpu(softnet_data, tcpu).input_queue_head -
		      rflow->last_qtail)) >= 0)) {
			tcpu = next_cpu;
			rflow = set_rps_cpu(dev, skb, rflow, next_cpu);
		}

		if (tcpu < nr_cpu_ids && cpu_online(tcpu)) {
			*rflowp = rflow;
			cpu = tcpu;
			goto done;
		}
	}

try_rps:

	if (map) {
		tcpu = map->cpus[reciprocal_scale(hash, map->len)];
		if (cpu_online(tcpu)) {
			cpu = tcpu;
			goto done;
		}
	}

done:
	return cpu;
}
```


# flow table 大小设置
```c
==== RFS Configuration

RFS is only available if the kconfig symbol CONFIG_RPS is enabled (on
by default for SMP). The functionality remains disabled until explicitly
configured. The number of entries in the global flow table is set through:

 /proc/sys/net/core/rps_sock_flow_entries

The number of entries in the per-queue flow table are set through:

 /sys/class/net/<dev>/queues/rx-<n>/rps_flow_cnt

== Suggested Configuration

Both of these need to be set before RFS is enabled for a receive queue.
Values for both are rounded up to the nearest power of two. The
suggested flow count depends on the expected number of active connections
at any given time, which may be significantly less than the number of open
connections. We have found that a value of 32768 for rps_sock_flow_entries
works fairly well on a moderately loaded server.

For a single queue device, the rps_flow_cnt value for the single queue
would normally be configured to the same value as rps_sock_flow_entries.
For a multi-queue device, the rps_flow_cnt for each queue might be
configured as rps_sock_flow_entries / N, where N is the number of
queues. So for instance, if rps_sock_flow_entries is set to 32768 and there
are 16 configured receive queues, rps_flow_cnt for each queue might be
configured as 2048.

```

