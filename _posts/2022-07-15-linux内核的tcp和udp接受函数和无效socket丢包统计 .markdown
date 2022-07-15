```text


tcp_v4_rcv  {  // tcp的接函数

lookup:
	sk = __inet_lookup_skb(&tcp_hashinfo, skb, __tcp_hdrlen(th), th->source,
			       th->dest, sdif, &refcounted);
	if (!sk)
		goto no_tcp_socket;

no_tcp_socket:
	drop_reason = SKB_DROP_REASON_NO_SOCKET;        //  丢包原因
	if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb))
		goto discard_it;

	tcp_v4_fill_cb(skb, iph, th);

	if (tcp_checksum_complete(skb)) {
csum_error:
		drop_reason = SKB_DROP_REASON_TCP_CSUM;
		trace_tcp_bad_csum(skb);
		__TCP_INC_STATS(net, TCP_MIB_CSUMERRORS);
bad_packet:
		__TCP_INC_STATS(net, TCP_MIB_INERRS);
	} else {
		tcp_v4_send_reset(NULL, skb);       // 没有socket就发送reset
                       __TCP_INC_STATS(net, TCP_MIB_OUTRSTS);   //  tcp_v4_send_reset 会统计发送socket的数量
	} 

discard_it:
	/* Discard frame. */
	kfree_skb_reason(skb, drop_reason);
	return 0;

}



int udp_rcv(struct sk_buff *skb)   // udp的接收函数
{
	return __udp4_lib_rcv(skb, &udp_table, IPPROTO_UDP);
}


int __udp4_lib_rcv(struct sk_buff *skb, struct udp_table *udptable,
		   int proto)



	drop_reason = SKB_DROP_REASON_NO_SOCKET;                            /// 丢包原因
	__UDP_INC_STATS(net, UDP_MIB_NOPORTS, proto == IPPROTO_UDPLITE);    /// 记录  UDP_MIB_NOPORTS 丢包
	icmp_send(skb, ICMP_DEST_UNREACH, ICMP_PORT_UNREACH, 0);

	/*
	 * Hmm.  We got an UDP packet to a port to which we
	 * don't wanna listen.  Ignore it.
	 */
	kfree_skb_reason(skb, drop_reason);



/**
 *	kfree_skb_reason - free an sk_buff with special reason
 *	@skb: buffer to free
 *	@reason: reason why this skb is dropped
 *
 *	Drop a reference to the buffer and free it if the usage count has
 *	hit zero. Meanwhile, pass the drop reason to 'kfree_skb'
 *	tracepoint.
 */
void kfree_skb_reason(struct sk_buff *skb, enum skb_drop_reason reason)
{
	if (!skb_unref(skb))
		return;

	trace_kfree_skb(skb, __builtin_return_address(0), reason);
	__kfree_skb(skb);
}
EXPORT_SYMBOL(kfree_skb_reason);



统计对应 cat /proc/net/snmp 输出里面的 OutRsts 和 NoPorts 的指标吧
  

```
