```text
PF_PACKET  往是专门用于抓包的，往系统网络层注册一个协议。然后所有的往外发的包和进来的包都会调到http://lxr.linux.no/linux+v3.4/net/packet/af_packet.c 这个文件里面 的packet_rcv() 函数，

其中 outgoing方向（出去的包）会在 dev_queue_xmit_nit里面遍历 ptype_all 链表进行所有网络协议处理的时候调用到packet_rcv 。
incoming  方向（从外面其他机器进来的包会在 netif_receive_skb 函数里面同样办法遍历 ptype_all进行处理的时候调用到packet_rcv 。


在之前的一篇文章
http://gmd20.blog.163.com/blog/static/16843923201011194136744/
里面有分析过了，但太久了又忘了出去方向的包是怎么又被发到PF_PACKET  的来了。唉！看看下面的吧，希望以后不会再次忘记了。



int dev_queue_xmit(struct sk_buff *skb)
2213                        if (!netif_tx_queue_stopped(txq)) {
2214                                rc = dev_hard_start_xmit(skb, dev, txq);
 
--------------------------------------

1938int dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev,
1939                        struct netdev_queue *txq)
1940{
1941        const struct net_device_ops *ops = dev->netdev_ops;
1942        int rc = NETDEV_TX_OK;
1943
1944        if (likely(!skb->next)) {
1945                if (!list_empty(&ptype_all))
1946                        dev_queue_xmit_nit(skb, dev);


-----------------------------------------------------

1500/*
1501 *      Support routine. Sends outgoing frames to any network
1502 *      taps currently in use.
1503 */
1504
1505static void dev_queue_xmit_nit(struct sk_buff *skb, struct net_device *dev)
1506{
1507        struct packet_type *ptype;
1508
1509#ifdef CONFIG_NET_CLS_ACT
1510        if (!(skb->tstamp.tv64 && (G_TC_FROM(skb->tc_verd) & AT_INGRESS)))
1511                net_timestamp_set(skb);
1512#else
1513        net_timestamp_set(skb);
1514#endif
1515
1516        rcu_read_lock();
1517        list_for_each_entry_rcu(ptype, &ptype_all, list) {
1518                /* Never send packets back to the socket
1519                 * they originated from - MvS (miquels@drinkel.ow.org)
1520                 */
1521                if ((ptype->dev == dev || !ptype->dev) &&
1522                    (ptype->af_packet_priv == NULL ||
1523                     (struct sock *)ptype->af_packet_priv != skb->sk)) {
1524                        struct sk_buff *skb2 = skb_clone(skb, GFP_ATOMIC);
1525                        if (!skb2)
1526                                break;
1527
1528                        /* skb->nh should be correctly
1529                           set by sender, so that the second statement is
1530                           just protection against buggy protocols.
1531                         */
1532                        skb_reset_mac_header(skb2);
1533
1534                        if (skb_network_header(skb2) < skb2->data ||
1535                            skb2->network_header > skb2->tail) {
1536                                if (net_ratelimit())
1537                                        printk(KERN_CRIT "protocol %04x is "
1538                                               "buggy, dev %s\n",
1539                                               ntohs(skb2->protocol),
1540                                               dev->name);
1541                                skb_reset_network_header(skb2);
1542                        }
1543
1544                        skb2->transport_header = skb2->network_header;
1545                        skb2->pkt_type = PACKET_OUTGOING;
1546                        ptype->func(skb2, skb->dev, ptype, skb->dev);  ///调用PF_PACKET协议注册的函数 packet_rcv())
1547                }
1548        }
1549        rcu_read_unlock();
1550}

--------------------------------
```
