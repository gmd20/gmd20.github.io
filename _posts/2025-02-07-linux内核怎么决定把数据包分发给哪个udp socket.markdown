```c
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
if (sockfd < 0) {
    perror("socket");
    exit(1);
}

struct sockaddr_in local_addr;
memset(&local_addr, 0, sizeof(local_addr));
local_addr.sin_family = AF_INET;
local_addr.sin_addr.s_addr = htonl(INADDR_ANY); // 监听所有接口, 或指定一个单播IP
local_addr.sin_port = htons(YOUR_PORT);  // 你要监听的端口

if (bind(sockfd, (struct sockaddr *)&local_addr, sizeof(local_addr)) < 0) {
    perror("bind");
    exit(1);
}

struct ip_mreq mreq;
mreq.imr_multiaddr.s_addr = inet_addr("YOUR_MULTICAST_GROUP_IP"); // 组播组IP
mreq.imr_interface.s_addr = htonl(INADDR_ANY); // 监听所有接口，或指定一个接口的IP

if (setsockopt(sockfd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq)) < 0) {
    perror("setsockopt (IP_ADD_MEMBERSHIP)");
    exit(1);
}
```

这样的socket ，绑定了 “0.0.0.0”的地址来接受 组播，到底能不能同时收到 组播和单播的数据呢。 让AI解释，感觉也不是很确定。



让AI 提示，应该是对应这个内核源码
```c
/* UDP is nearly always wildcards out the wazoo, it makes no sense to try
 * harder than this. -DaveM
 */
struct sock *__udp4_lib_lookup(const struct net *net, __be32 saddr,
		__be16 sport, __be32 daddr, __be16 dport, int dif,
		int sdif, struct udp_table *udptable, struct sk_buff *skb)
{
	unsigned short hnum = ntohs(dport);
	struct udp_hslot *hslot2;
	struct sock *result, *sk;
	unsigned int hash2;

	hash2 = ipv4_portaddr_hash(net, daddr, hnum);
	hslot2 = udp_hashslot2(udptable, hash2);

	if (udp_has_hash4(hslot2)) {
		result = udp4_lib_lookup4(net, saddr, sport, daddr, hnum,
					  dif, sdif, udptable);
		if (result) /* udp4_lib_lookup4 return sk or NULL */
			return result;
	}

	/* Lookup connected or non-wildcard socket */
	result = udp4_lib_lookup2(net, saddr, sport,
				  daddr, hnum, dif, sdif,
				  hslot2, skb);
	if (!IS_ERR_OR_NULL(result) && result->sk_state == TCP_ESTABLISHED)
		goto done;

	/* Lookup redirect from BPF */
	if (static_branch_unlikely(&bpf_sk_lookup_enabled) &&
	    udptable == net->ipv4.udp_table) {
		sk = inet_lookup_run_sk_lookup(net, IPPROTO_UDP, skb, sizeof(struct udphdr),
					       saddr, sport, daddr, hnum, dif,
					       udp_ehashfn);
		if (sk) {
			result = sk;
			goto done;
		}
	}

	/* Got non-wildcard socket or error on first lookup */
	if (result)
		goto done;

	/* Lookup wildcard sockets */
	hash2 = ipv4_portaddr_hash(net, htonl(INADDR_ANY), hnum);
	hslot2 = udp_hashslot2(udptable, hash2);

	result = udp4_lib_lookup2(net, saddr, sport,
				  htonl(INADDR_ANY), hnum, dif, sdif,
				  hslot2, skb);
done:
	if (IS_ERR(result))
		return NULL;
	return result;
}
EXPORT_SYMBOL_GPL(__udp4_lib_lookup);


/*
 *	All we need to do is get the socket, and then do a checksum.
 */

int __udp4_lib_rcv(struct sk_buff *skb, struct udp_table *udptable,
		   int proto)
{
	struct sock *sk = NULL;
	struct udphdr *uh;
	unsigned short ulen;
	struct rtable *rt = skb_rtable(skb);
	__be32 saddr, daddr;
	struct net *net = dev_net(skb->dev);
	bool refcounted;
	int drop_reason;

	drop_reason = SKB_DROP_REASON_NOT_SPECIFIED;

	/*
	 *  Validate the packet.
	 */
	if (!pskb_may_pull(skb, sizeof(struct udphdr)))
		goto drop;		/* No space for header. */

	uh   = udp_hdr(skb);
	ulen = ntohs(uh->len);
	saddr = ip_hdr(skb)->saddr;
	daddr = ip_hdr(skb)->daddr;

	if (ulen > skb->len)
		goto short_packet;

	if (proto == IPPROTO_UDP) {
		/* UDP validates ulen. */
		if (ulen < sizeof(*uh) || pskb_trim_rcsum(skb, ulen))
			goto short_packet;
		uh = udp_hdr(skb);
	}

	if (udp4_csum_init(skb, uh, proto))
		goto csum_error;

	sk = inet_steal_sock(net, skb, sizeof(struct udphdr), saddr, uh->source, daddr, uh->dest,
			     &refcounted, udp_ehashfn);
	if (IS_ERR(sk))
		goto no_sk;

	if (sk) {
		struct dst_entry *dst = skb_dst(skb);
		int ret;

		if (unlikely(rcu_dereference(sk->sk_rx_dst) != dst))
			udp_sk_rx_dst_set(sk, dst);

		ret = udp_unicast_rcv_skb(sk, skb, uh);
		if (refcounted)
			sock_put(sk);
		return ret;
	}

	if (rt->rt_flags & (RTCF_BROADCAST|RTCF_MULTICAST))
		return __udp4_lib_mcast_deliver(net, skb, uh,
						saddr, daddr, udptable, proto);

	sk = __udp4_lib_lookup_skb(skb, uh->source, uh->dest, udptable);
	if (sk)
		return udp_unicast_rcv_skb(sk, skb, uh);
no_sk:
	if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb))
		goto drop;
	nf_reset_ct(skb);

	/* No socket. Drop packet silently, if checksum is wrong */
	if (udp_lib_checksum_complete(skb))
		goto csum_error;

	drop_reason = SKB_DROP_REASON_NO_SOCKET;
	__UDP_INC_STATS(net, UDP_MIB_NOPORTS, proto == IPPROTO_UDPLITE);
	icmp_send(skb, ICMP_DEST_UNREACH, ICMP_PORT_UNREACH, 0);

	/*
	 * Hmm.  We got an UDP packet to a port to which we
	 * don't wanna listen.  Ignore it.
	 */
	sk_skb_reason_drop(sk, skb, drop_reason);
	return 0;

short_packet:
	drop_reason = SKB_DROP_REASON_PKT_TOO_SMALL;
	net_dbg_ratelimited("UDP%s: short packet: From %pI4:%u %d/%d to %pI4:%u\n",
			    proto == IPPROTO_UDPLITE ? "Lite" : "",
			    &saddr, ntohs(uh->source),
			    ulen, skb->len,
			    &daddr, ntohs(uh->dest));
	goto drop;

csum_error:
	/*
	 * RFC1122: OK.  Discards the bad packet silently (as far as
	 * the network is concerned, anyway) as per 4.1.3.4 (MUST).
	 */
	drop_reason = SKB_DROP_REASON_UDP_CSUM;
	net_dbg_ratelimited("UDP%s: bad checksum. From %pI4:%u to %pI4:%u ulen %d\n",
			    proto == IPPROTO_UDPLITE ? "Lite" : "",
			    &saddr, ntohs(uh->source), &daddr, ntohs(uh->dest),
			    ulen);
	__UDP_INC_STATS(net, UDP_MIB_CSUMERRORS, proto == IPPROTO_UDPLITE);
drop:
	__UDP_INC_STATS(net, UDP_MIB_INERRORS, proto == IPPROTO_UDPLITE);
	sk_skb_reason_drop(sk, skb, drop_reason);
	return 0;
}


static inline bool __udp_is_mcast_sock(struct net *net, const struct sock *sk,
				       __be16 loc_port, __be32 loc_addr,
				       __be16 rmt_port, __be32 rmt_addr,
				       int dif, int sdif, unsigned short hnum)
{
	const struct inet_sock *inet = inet_sk(sk);

	if (!net_eq(sock_net(sk), net) ||
	    udp_sk(sk)->udp_port_hash != hnum ||
	    (inet->inet_daddr && inet->inet_daddr != rmt_addr) ||
	    (inet->inet_dport != rmt_port && inet->inet_dport) ||
	    (inet->inet_rcv_saddr && inet->inet_rcv_saddr != loc_addr) ||
	    ipv6_only_sock(sk) ||
	    !udp_sk_bound_dev_eq(net, sk->sk_bound_dev_if, dif, sdif))
		return false;
	if (!ip_mc_sf_allow(sk, loc_addr, rmt_addr, dif, sdif))
		return false;
	return true;
}

/*
 * check if a multicast source filter allows delivery for a given <src,dst,intf>
 */
int ip_mc_sf_allow(const struct sock *sk, __be32 loc_addr, __be32 rmt_addr,
		   int dif, int sdif)
{
	const struct inet_sock *inet = inet_sk(sk);
	struct ip_mc_socklist *pmc;
	struct ip_sf_socklist *psl;
	int i;
	int ret;

	ret = 1;
	if (!ipv4_is_multicast(loc_addr))
		goto out;

	rcu_read_lock();
	for_each_pmc_rcu(inet, pmc) {
		if (pmc->multi.imr_multiaddr.s_addr == loc_addr &&
		    (pmc->multi.imr_ifindex == dif ||
		     (sdif && pmc->multi.imr_ifindex == sdif)))
			break;
	}
	ret = inet_test_bit(MC_ALL, sk);
	if (!pmc)
		goto unlock;
	psl = rcu_dereference(pmc->sflist);
	ret = (pmc->sfmode == MCAST_EXCLUDE);
	if (!psl)
		goto unlock;

	for (i = 0; i < psl->sl_count; i++) {
		if (psl->sl_addr[i] == rmt_addr)
			break;
	}
	ret = 0;
	if (pmc->sfmode == MCAST_INCLUDE && i >= psl->sl_count)
		goto unlock;
	if (pmc->sfmode == MCAST_EXCLUDE && i < psl->sl_count)
		goto unlock;
	ret = 1;
unlock:
	rcu_read_unlock();
out:
	return ret;
}

```


这里组播包和 单播包的 socket查找应该是  __udp4_lib_mcast_deliver  和 __udp4_lib_lookup_skb了。   


但这里还看到 一个  bpf 来自定义选择的socket的。
```c
	/* Lookup redirect from BPF */
	if (static_branch_unlikely(&bpf_sk_lookup_enabled) &&
	    udptable == net->ipv4.udp_table) {
		sk = inet_lookup_run_sk_lookup(net, IPPROTO_UDP, skb, sizeof(struct udphdr),
					       saddr, sport, daddr, hnum, dif,
					       udp_ehashfn);
		if (sk) {
			result = sk;
			goto done;
		}
	}

struct sock *inet_lookup_run_sk_lookup(const struct net *net,
				       int protocol,
				       struct sk_buff *skb, int doff,
				       __be32 saddr, __be16 sport,
				       __be32 daddr, u16 hnum, const int dif,
				       inet_ehashfn_t *ehashfn)
{
	struct sock *sk, *reuse_sk;
	bool no_reuseport;

	no_reuseport = bpf_sk_lookup_run_v4(net, protocol, saddr, sport,
					    daddr, hnum, dif, &sk);
	if (no_reuseport || IS_ERR_OR_NULL(sk))
		return sk;

	reuse_sk = inet_lookup_reuseport(net, sk, skb, doff, saddr, sport, daddr, hnum,
					 ehashfn);
	if (reuse_sk)
		sk = reuse_sk;
	return sk;
}


static inline bool bpf_sk_lookup_run_v4(const struct net *net, int protocol,
					const __be32 saddr, const __be16 sport,
					const __be32 daddr, const u16 dport,
					const int ifindex, struct sock **psk)
{
	struct bpf_prog_array *run_array;
	struct sock *selected_sk = NULL;
	bool no_reuseport = false;

	rcu_read_lock();
	run_array = rcu_dereference(net->bpf.run_array[NETNS_BPF_SK_LOOKUP]);
	if (run_array) {
		struct bpf_sk_lookup_kern ctx = {
			.family		= AF_INET,
			.protocol	= protocol,
			.v4.saddr	= saddr,
			.v4.daddr	= daddr,
			.sport		= sport,
			.dport		= dport,
			.ingress_ifindex	= ifindex,
		};
		u32 act;

		act = BPF_PROG_SK_LOOKUP_RUN_ARRAY(run_array, ctx, bpf_prog_run);
		if (act == SK_PASS) {
			selected_sk = ctx.selected_sk;
			no_reuseport = ctx.no_reuseport;
		} else {
			selected_sk = ERR_PTR(-ECONNREFUSED);
		}
	}
	rcu_read_unlock();
	*psk = selected_sk;
	return no_reuseport;
}

```

最初的需求应该是一个 socket接收大量不同ip 和端口之类的需求吧，比较灵活的控制socket查找的逻辑。
不过感觉还是比较麻烦，要bpf里面自己管理sockmap，详细可以参考下面这几篇文章

“BPF sk_lookup program”
https://www.kernel.org/doc/html/latest/bpf/prog_sk_lookup.html

“[译] Socket listen 多地址需求与 SK_LOOKUP BPF 的诞生”
https://arthurchiao.art/blog/birth-of-sk-lookup-bpf-zh/

“It's crowded in here”
https://blog.cloudflare.com/its-crowded-in-here/

“Pidfd and Socket-lookup BPF (SK_LOOKUP) Illustrated (2022)”
https://arthurchiao.art/blog/pidfd-and-socket-lookup-bpf-illustrated/
















