发现一个奇怪的事情，通过下面这两个命令发现内核里面很多ARP表的数据STALE和FAILED的记录（incomplete）的记录，
```text
ip -s neigh
arp -an 
```
失效时间已经远远大于内核参数控制失效时间，但记录还保留在表里面。
```text
/proc/sys/net/ipv4/neigh/default/base_reachable_time 
/proc/sys/net/ipv4/neigh/default/gc_stale_time
/proc/sys/net/ipv4/neigh/default/gc_interval
```

通过 ioctl SIOCGARP 还是能过获取到ip对应的mac地址的，只是ATF_COM标识为表示为icomplete项而已。
查看了一下网上确实也有人发现这个现象，几天都不清理也是有可能的
https://serverfault.com/questions/765380/when-do-stale-arp-entries-become-failed-when-never-used

对应的代码函数在这里
https://github.com/torvalds/linux/blob/345671ea0f9258f410eb057b9ced9cefbbe5dc78/net/core/neighbour.c
```c
static void neigh_periodic_work(struct work_struct *work)
{
	struct neigh_table *tbl = container_of(work, struct neigh_table, gc_work.work);
	struct neighbour *n;
	struct neighbour __rcu **np;
	unsigned int i;
	struct neigh_hash_table *nht;

	NEIGH_CACHE_STAT_INC(tbl, periodic_gc_runs);

	write_lock_bh(&tbl->lock);
	nht = rcu_dereference_protected(tbl->nht,
					lockdep_is_held(&tbl->lock));

	/*
	 *	periodically recompute ReachableTime from random function
	 */

	if (time_after(jiffies, tbl->last_rand + 300 * HZ)) {
		struct neigh_parms *p;
		tbl->last_rand = jiffies;
		list_for_each_entry(p, &tbl->parms_list, list)
			p->reachable_time =
				neigh_rand_reach_time(NEIGH_VAR(p, BASE_REACHABLE_TIME));
	}

	if (atomic_read(&tbl->entries) < tbl->gc_thresh1)   //如果这个值没有满足，就永远不会扫描arp表清理失效的记录。
		goto out;

	for (i = 0 ; i < (1 << nht->hash_shift); i++) {   // 扫描清理 arp的hash表
		np = &nht->hash_buckets[i];

		while ((n = rcu_dereference_protected(*np,
				lockdep_is_held(&tbl->lock))) != NULL) {
			unsigned int state;

			write_lock(&n->lock);

			state = n->nud_state;
			if ((state & (NUD_PERMANENT | NUD_IN_TIMER)) ||
			    (n->flags & NTF_EXT_LEARNED)) {
				write_unlock(&n->lock);
				goto next_elt;
			}

			if (time_before(n->used, n->confirmed))
				n->used = n->confirmed;

			if (refcount_read(&n->refcnt) == 1 &&
			    (state == NUD_FAILED ||
			     time_after(jiffies, n->used + NEIGH_VAR(n->parms, GC_STALETIME)))) {
				*np = n->next;
				n->dead = 1;
				write_unlock(&n->lock);
				neigh_cleanup_and_release(n);
				continue;
			}
			write_unlock(&n->lock);

next_elt:
			np = &n->next;
		}
		/*
		 * It's fine to release lock here, even if hash table
		 * grows while we are preempted.
		 */
		write_unlock_bh(&tbl->lock);
		cond_resched();
		write_lock_bh(&tbl->lock);
		nht = rcu_dereference_protected(tbl->nht,
						lockdep_is_held(&tbl->lock));
	}
out:
	/* Cycle through all hash buckets every BASE_REACHABLE_TIME/2 ticks.
	 * ARP entry timeouts range from 1/2 BASE_REACHABLE_TIME to 3/2
	 * BASE_REACHABLE_TIME.
	 */
	queue_delayed_work(system_power_efficient_wq, &tbl->gc_work,
			      NEIGH_VAR(&tbl->parms, BASE_REACHABLE_TIME) >> 1);
	write_unlock_bh(&tbl->lock);
}
```


其实关键就是这个 /proc/sys/net/ipv4/neigh/default/gc_thresh1   参数的设置，如果这个值设置的比较大，只要表里面的总记录的数目小于这个值
gc的工作就没做，直接跳过了。这个地方有点奇怪，其实可以甚至一下大于某个固定值比如一两个小时就强制扫描一下也是完全不会影响性能的吧。

知道了这点，要强制扫描清理一下表也就很简单了， 
```text
echo 128 > /proc/sys/net/ipv4/neigh/default/gc_thresh1
echo 56 > /proc/sys/net/ipv4/neigh/default/gc_thresh1
```
设置一个比当前的arp表的记录少的值，gc就马上启动清理完过时太久的数据了。
可以参考官方文档的说明
http://man7.org/linux/man-pages/man7/arp.7.html 

其实一般这个不清理也没有太大影响， ping 这个ip或者发起其他网络操作的时候，内核自动发起arp检测来重新核对arp记录的有效性的。
