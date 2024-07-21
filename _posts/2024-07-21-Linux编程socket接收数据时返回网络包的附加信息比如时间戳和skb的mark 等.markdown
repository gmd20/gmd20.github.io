socket设置了SO_TIMESTAMP后，会在接收数据时通过put_cmsg往控制头上面写入信息吧，比如接收数据包的时间戳和skb的mark还有比如vlan id等 都是通过这个方式返回。

```c
/*
 *	This is meant for all protocols to use and covers goings on
 *	at the socket level. Everything here is generic.
 */

int sk_setsockopt(struct sock *sk, int level, int optname,
		  sockptr_t optval, unsigned int optlen)
{
                switch (optname) {
	case SO_TIMESTAMP_OLD:
	case SO_TIMESTAMP_NEW:
	case SO_TIMESTAMPNS_OLD:
	case SO_TIMESTAMPNS_NEW:
		sock_set_timestamp(sk, optname, valbool);
		break;
               }
{

void sock_set_timestamp(struct sock *sk, int optname, bool valbool)
{
	switch (optname) {
	case SO_TIMESTAMP_OLD:
		__sock_set_timestamps(sk, valbool, false, false);
		break;
	case SO_TIMESTAMP_NEW:
		__sock_set_timestamps(sk, valbool, true, false);
		break;
	case SO_TIMESTAMPNS_OLD:
		__sock_set_timestamps(sk, valbool, false, true);
		break;
	case SO_TIMESTAMPNS_NEW:
		__sock_set_timestamps(sk, valbool, true, true);
		break;
	}
}
    
  
 int enable = 1;
    if (setsockopt(sock, SOL_SOCKET, SO_TIMESTAMP, &enable, sizeof(enable))) {
    }

     struct msghdr m;
     int result = recvmsg(socket_info.sockfd_, &m, 0);
    struct cmsghdr* cmsg = CMSG_FIRSTHDR(&m);
    while (cmsg != NULL) {
        if ((cmsg->cmsg_level == SOL_SOCKET) &&
            (cmsg->cmsg_type  == SCM_TIMESTAMP)) {
               struct timeval cmsg_time;
            memcpy(&cmsg_time, CMSG_DATA(cmsg), sizeof(cmsg_time));
     }


static void __sock_set_timestamps(struct sock *sk, bool val, bool new, bool ns)
{
	if (val)  {
		sock_valbool_flag(sk, SOCK_TSTAMP_NEW, new);
		sock_valbool_flag(sk, SOCK_RCVTSTAMPNS, ns);
		sock_set_flag(sk, SOCK_RCVTSTAMP);
		sock_enable_timestamp(sk, SOCK_TIMESTAMP);
	} else {
		sock_reset_flag(sk, SOCK_RCVTSTAMP);
		sock_reset_flag(sk, SOCK_RCVTSTAMPNS);
	}
}


void __sock_recv_cmsgs(struct msghdr *msg, struct sock *sk,
		       struct sk_buff *skb)
{
	sock_recv_timestamp(msg, sk, skb);
	sock_recv_drops(msg, sk, skb);
	sock_recv_mark(msg, sk, skb);
}
EXPORT_SYMBOL_GPL(__sock_recv_cmsgs);

static inline void
sock_recv_timestamp(struct msghdr *msg, struct sock *sk, struct sk_buff *skb)
{
	struct skb_shared_hwtstamps *hwtstamps = skb_hwtstamps(skb);
	u32 tsflags = READ_ONCE(sk->sk_tsflags);
	ktime_t kt = skb->tstamp;
	/*
	 * generate control messages if
	 * - receive time stamping in software requested
	 * - software time stamp available and wanted
	 * - hardware time stamps available and wanted
	 */
	if (sock_flag(sk, SOCK_RCVTSTAMP) ||
	    (tsflags & SOF_TIMESTAMPING_RX_SOFTWARE) ||
	    (kt && tsflags & SOF_TIMESTAMPING_SOFTWARE) ||
	    (hwtstamps->hwtstamp &&
	     (tsflags & SOF_TIMESTAMPING_RAW_HARDWARE)))
		__sock_recv_timestamp(msg, sk, skb);
	else
		sock_write_timestamp(sk, kt);

	if (sock_flag(sk, SOCK_WIFI_STATUS) && skb_wifi_acked_valid(skb))
		__sock_recv_wifi_status(msg, sk, skb);
}



/*
 * called from sock_recv_timestamp() if sock_flag(sk, SOCK_RCVTSTAMP)
 */
void __sock_recv_timestamp(struct msghdr *msg, struct sock *sk,
	struct sk_buff *skb)
{
	int need_software_tstamp = sock_flag(sk, SOCK_RCVTSTAMP);
	int new_tstamp = sock_flag(sk, SOCK_TSTAMP_NEW);
	struct scm_timestamping_internal tss;
	int empty = 1, false_tstamp = 0;
	struct skb_shared_hwtstamps *shhwtstamps =
		skb_hwtstamps(skb);
	int if_index;
	ktime_t hwtstamp;
	u32 tsflags;

	/* Race occurred between timestamp enabling and packet
	   receiving.  Fill in the current time for now. */
	if (need_software_tstamp && skb->tstamp == 0) {
		__net_timestamp(skb);
		false_tstamp = 1;
	}

	if (need_software_tstamp) {
		if (!sock_flag(sk, SOCK_RCVTSTAMPNS)) {
			if (new_tstamp) {
				struct __kernel_sock_timeval tv;

				skb_get_new_timestamp(skb, &tv);
				put_cmsg(msg, SOL_SOCKET, SO_TIMESTAMP_NEW,
					 sizeof(tv), &tv);
			} else {
				struct __kernel_old_timeval tv;

				skb_get_timestamp(skb, &tv);
				put_cmsg(msg, SOL_SOCKET, SO_TIMESTAMP_OLD,
					 sizeof(tv), &tv);
			}
		} else {
			if (new_tstamp) {
				struct __kernel_timespec ts;

				skb_get_new_timestampns(skb, &ts);
				put_cmsg(msg, SOL_SOCKET, SO_TIMESTAMPNS_NEW,
					 sizeof(ts), &ts);
			} else {
				struct __kernel_old_timespec ts;

				skb_get_timestampns(skb, &ts);
				put_cmsg(msg, SOL_SOCKET, SO_TIMESTAMPNS_OLD,
					 sizeof(ts), &ts);
			}
		}
	}

	memset(&tss, 0, sizeof(tss));
	tsflags = READ_ONCE(sk->sk_tsflags);
	if ((tsflags & SOF_TIMESTAMPING_SOFTWARE) &&
	    ktime_to_timespec64_cond(skb->tstamp, tss.ts + 0))
		empty = 0;
	if (shhwtstamps &&
	    (tsflags & SOF_TIMESTAMPING_RAW_HARDWARE) &&
	    !skb_is_swtx_tstamp(skb, false_tstamp)) {
		if_index = 0;
		if (skb_shinfo(skb)->tx_flags & SKBTX_HW_TSTAMP_NETDEV)
			hwtstamp = get_timestamp(sk, skb, &if_index);
		else
			hwtstamp = shhwtstamps->hwtstamp;

		if (tsflags & SOF_TIMESTAMPING_BIND_PHC)
			hwtstamp = ptp_convert_timestamp(&hwtstamp,
							 READ_ONCE(sk->sk_bind_phc));

		if (ktime_to_timespec64_cond(hwtstamp, tss.ts + 2)) {
			empty = 0;

			if ((tsflags & SOF_TIMESTAMPING_OPT_PKTINFO) &&
			    !skb_is_err_queue(skb))
				put_ts_pktinfo(msg, skb, if_index);
		}
	}
	if (!empty) {
		if (sock_flag(sk, SOCK_TSTAMP_NEW))
			put_cmsg_scm_timestamping64(msg, &tss);
		else
			put_cmsg_scm_timestamping(msg, &tss);

		if (skb_is_err_queue(skb) && skb->len &&
		    SKB_EXT_ERR(skb)->opt_stats)
			put_cmsg(msg, SOL_SOCKET, SCM_TIMESTAMPING_OPT_STATS,
				 skb->len, skb->data);
	}
}
EXPORT_SYMBOL_GPL(__sock_recv_timestamp);






static void sock_recv_mark(struct msghdr *msg, struct sock *sk,
			   struct sk_buff *skb)
{
	if (sock_flag(sk, SOCK_RCVMARK) && skb) {
		/* We must use a bounce buffer for CONFIG_HARDENED_USERCOPY=y */
		__u32 mark = skb->mark;

		put_cmsg(msg, SOL_SOCKET, SO_MARK, sizeof(__u32), &mark);
	}
}



```
