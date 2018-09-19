```text

netconsole是一个内核模块的远程console的实现。发现这个netpoll模块的代码还是很不错的，是学习怎么使用“skb队列”和做虚拟网卡是怎么利用“物理网卡”的

一个不错的参考。注意skb的入队和出队操作（skb_dequeue skb_queue_head）等，锁的使用，物理网卡状态的管理和检测（netif_running 

netif_device_present 等）， net_device结构的 ndo_start_xmit操作的调用，繁忙时的poll操作调用和演示发送，发送错误的重新入队等。

在上家公司时，那个虚拟网卡的实现的队列管理和这个非常的像啊，难得抄的这个？？呵呵





http://lxr.linux.no/linux+v3.1.6/net/core/netpoll.c#L57

 57static void queue_process(struct work_struct *work)
  58{
  59        struct netpoll_info *npinfo =
  60                container_of(work, struct netpoll_info, tx_work.work);
  61        struct sk_buff *skb;
  62        unsigned long flags;
  63
  64        while ((skb = skb_dequeue(&npinfo->txq))) {    ///////////从队列拿出skb 
  65                struct net_device *dev = skb->dev;
  66                const struct net_device_ops *ops = dev->netdev_ops;
  67                struct netdev_queue *txq;
  68
  69                if (!netif_device_present(dev) || !netif_running(dev)) {
  70                        __kfree_skb(skb);
  71                        continue;
  72                }
  73
  74                txq = netdev_get_tx_queue(dev, skb_get_queue_mapping(skb));
  75
  76                local_irq_save(flags);
  77                __netif_tx_lock(txq, smp_processor_id());
  78                if (netif_tx_queue_frozen_or_stopped(txq) ||
  79                    ops->ndo_start_xmit(skb, dev) != NETDEV_TX_OK) {
  80                        skb_queue_head(&npinfo->txq, skb);         //发送不出去的时候，把包前加到队列的前面，再次重试
  81                        __netif_tx_unlock(txq);
  82                        local_irq_restore(flags);
  83
  84                        schedule_delayed_work(&npinfo->tx_work, HZ/10);  //演示再调用queue_process 回来继续处理
  85                        return;
  86                }
  87                __netif_tx_unlock(txq);
  88                local_irq_restore(flags);
  89        }
  90}








 180static void netpoll_poll_dev(struct net_device *dev)
 181{
 182        const struct net_device_ops *ops;
 183
 184        if (!dev || !netif_running(dev))
 185                return;
 186
 187        ops = dev->netdev_ops;
 188        if (!ops->ndo_poll_controller)
 189                return;
 190
 191        /* Process pending work on NIC */
 192        ops->ndo_poll_controller(dev);  ////////////网卡的poll操作
 193
 194        poll_napi(dev);   ///////////new api的poll ？ 
 195
 196        if (dev->priv_flags & IFF_SLAVE) {
 197                if (dev->npinfo) {
 198                        struct net_device *bond_dev = dev->master;
 199                        struct sk_buff *skb;
 200                        while ((skb = skb_dequeue(&dev->npinfo->arp_tx))) {
 201                                skb->dev = bond_dev;
 202                                skb_queue_tail(&bond_dev->npinfo->arp_tx, skb);
 203                        }
 204                }
 205        }
 206
 207        service_arp_queue(dev->npinfo);
 208
 209        zap_completion_queue();
 210}
 211












 293void netpoll_send_skb_on_dev(struct netpoll *np, struct sk_buff *skb,
 294                             struct net_device *dev)
 295{
 296        int status = NETDEV_TX_BUSY;
 297        unsigned long tries;
 298        const struct net_device_ops *ops = dev->netdev_ops;
 299        /* It is up to the caller to keep npinfo alive. */
 300        struct netpoll_info *npinfo = np->dev->npinfo;
 301
 302        if (!npinfo || !netif_running(dev) || !netif_device_present(dev)) {  ////检查实际物理网卡的状态
 303                __kfree_skb(skb);
 304                return;
 305        }
 306
 307        /* don't get messages out of order, and no recursion */
 308        if (skb_queue_len(&npinfo->txq) == 0 && !netpoll_owner_active(dev)) {
 309                struct netdev_queue *txq;
 310                unsigned long flags;
 311
 312                txq = netdev_get_tx_queue(dev, skb_get_queue_mapping(skb));
 313
 314                local_irq_save(flags);
 315                /* try until next clock tick */
 316                for (tries = jiffies_to_usecs(1)/USEC_PER_POLL;
 317                     tries > 0; --tries) {
 318                        if (__netif_tx_trylock(txq)) {
 319                                if (!netif_tx_queue_stopped(txq)) {
 320                                        status = ops->ndo_start_xmit(skb, dev);  ///调用物理设备的网卡的接口发送包
 321                                        if (status == NETDEV_TX_OK)
 322                                                txq_trans_update(txq);
 323                                }
 324                                __netif_tx_unlock(txq);
 325
 326                                if (status == NETDEV_TX_OK)
 327                                        break;
 328
 329                        }
 330
 331                        /* tickle device maybe there is some cleanup */
 332                        netpoll_poll_dev(np->dev);                       ///////掉过包发不出去，可能太多包要发了，调用new api的poll操作叫

物理网先把数据发出去，清理一下队列，可能下次就有机会发了
 333
 334                        udelay(USEC_PER_POLL);
 335                }
 336
 337                WARN_ONCE(!irqs_disabled(),
 338                        "netpoll_send_skb(): %s enabled interrupts in poll (%pF)\n",
 339                        dev->name, ops->ndo_start_xmit);
 340
 341                local_irq_restore(flags);
 342        }
 343
 344        if (status != NETDEV_TX_OK) {
 345                skb_queue_tail(&npinfo->txq, skb);       ////发送不成功，先把包加到队列的后面，待以后处理
 346                schedule_delayed_work(&npinfo->tx_work,0);
 347        }
 348}
 349EXPORT_SYMBOL(netpoll_send_skb_on_dev);
 350












///怎么构建skb并发送出去的

 351void netpoll_send_udp(struct netpoll *np, const char *msg, int len)
 352{
 353        int total_len, eth_len, ip_len, udp_len;
 354        struct sk_buff *skb;
 355        struct udphdr *udph;
 356        struct iphdr *iph;
 357        struct ethhdr *eth;
 358
 359        udp_len = len + sizeof(*udph);
 360        ip_len = eth_len = udp_len + sizeof(*iph);
 361        total_len = eth_len + ETH_HLEN + NET_IP_ALIGN;
 362
 363        skb = find_skb(np, total_len, total_len - len);
 364        if (!skb)
 365                return;
 366
 367        skb_copy_to_linear_data(skb, msg, len);
 368        skb->len += len;
 369
 370        skb_push(skb, sizeof(*udph));
 371        skb_reset_transport_header(skb);
 372        udph = udp_hdr(skb);
 373        udph->source = htons(np->local_port);
 374        udph->dest = htons(np->remote_port);
 375        udph->len = htons(udp_len);
 376        udph->check = 0;
 377        udph->check = csum_tcpudp_magic(np->local_ip,
 378                                        np->remote_ip,
 379                                        udp_len, IPPROTO_UDP,
 380                                        csum_partial(udph, udp_len, 0));
 381        if (udph->check == 0)
 382                udph->check = CSUM_MANGLED_0;
 383
 384        skb_push(skb, sizeof(*iph));
 385        skb_reset_network_header(skb);
 386        iph = ip_hdr(skb);
 387
 388        /* iph->version = 4; iph->ihl = 5; */
 389        put_unaligned(0x45, (unsigned char *)iph);
 390        iph->tos      = 0;
 391        put_unaligned(htons(ip_len), &(iph->tot_len));
 392        iph->id       = 0;
 393        iph->frag_off = 0;
 394        iph->ttl      = 64;
 395        iph->protocol = IPPROTO_UDP;
 396        iph->check    = 0;
 397        put_unaligned(np->local_ip, &(iph->saddr));
 398        put_unaligned(np->remote_ip, &(iph->daddr));
 399        iph->check    = ip_fast_csum((unsigned char *)iph, iph->ihl);
 400
 401        eth = (struct ethhdr *) skb_push(skb, ETH_HLEN);
 402        skb_reset_mac_header(skb);
 403        skb->protocol = eth->h_proto = htons(ETH_P_IP);
 404        memcpy(eth->h_source, np->dev->dev_addr, ETH_ALEN);
 405        memcpy(eth->h_dest, np->remote_mac, ETH_ALEN);
 406
 407        skb->dev = np->dev;
 408
 409        netpoll_send_skb(np, skb);
 410}
 411EXPORT_SYMBOL(netpoll_send_udp);









 413static void arp_reply(struct sk_buff *skb)
 414{
 415        struct netpoll_info *npinfo = skb->dev->npinfo;
 416        struct arphdr *arp;
 417        unsigned char *arp_ptr;
 418        int size, type = ARPOP_REPLY, ptype = ETH_P_ARP;
 419        __be32 sip, tip;
 420        unsigned char *sha;
 421        struct sk_buff *send_skb;
 422        struct netpoll *np, *tmp;
 423        unsigned long flags;
 424        int hits = 0;
 425
 426        if (list_empty(&npinfo->rx_np))
 427                return;
 428
 429        /* Before checking the packet, we do some early
 430           inspection whether this is interesting at all */
 431        spin_lock_irqsave(&npinfo->rx_lock, flags);
 432        list_for_each_entry_safe(np, tmp, &npinfo->rx_np, rx) {
 433                if (np->dev == skb->dev)
 434                        hits++;
 435        }
 436        spin_unlock_irqrestore(&npinfo->rx_lock, flags);
 437
 438        /* No netpoll struct is using this dev */
 439        if (!hits)
 440                return;
 441
 442        /* No arp on this interface */
 443        if (skb->dev->flags & IFF_NOARP)
 444                return;
 445
 446        if (!pskb_may_pull(skb, arp_hdr_len(skb->dev)))
 447                return;
 448
 449        skb_reset_network_header(skb);
 450        skb_reset_transport_header(skb);
 451        arp = arp_hdr(skb);
 452
 453        if ((arp->ar_hrd != htons(ARPHRD_ETHER) &&
 454             arp->ar_hrd != htons(ARPHRD_IEEE802)) ||
 455            arp->ar_pro != htons(ETH_P_IP) ||
 456            arp->ar_op != htons(ARPOP_REQUEST))
 457                return;
 458
 459        arp_ptr = (unsigned char *)(arp+1);
 460        /* save the location of the src hw addr */
 461        sha = arp_ptr;
 462        arp_ptr += skb->dev->addr_len;
 463        memcpy(&sip, arp_ptr, 4);
 464        arp_ptr += 4;
 465        /* If we actually cared about dst hw addr,
 466           it would get copied here */
 467        arp_ptr += skb->dev->addr_len;
 468        memcpy(&tip, arp_ptr, 4);
 469
 470        /* Should we ignore arp? */
 471        if (ipv4_is_loopback(tip) || ipv4_is_multicast(tip))
 472                return;
 473
 474        size = arp_hdr_len(skb->dev);
 475
 476        spin_lock_irqsave(&npinfo->rx_lock, flags);
 477        list_for_each_entry_safe(np, tmp, &npinfo->rx_np, rx) {
 478                if (tip != np->local_ip)
 479                        continue;
 480
 481                send_skb = find_skb(np, size + LL_ALLOCATED_SPACE(np->dev),
 482                                    LL_RESERVED_SPACE(np->dev));
 483                if (!send_skb)
 484                        continue;
 485
 486                skb_reset_network_header(send_skb);
 487                arp = (struct arphdr *) skb_put(send_skb, size);
 488                send_skb->dev = skb->dev;
 489                send_skb->protocol = htons(ETH_P_ARP);
 490
 491                /* Fill the device header for the ARP frame */
 492                if (dev_hard_header(send_skb, skb->dev, ptype,
 493                                    sha, np->dev->dev_addr,
 494                                    send_skb->len) < 0) {
 495                        kfree_skb(send_skb);
 496                        continue;
 497                }
 498
 499                /*
 500                 * Fill out the arp protocol part.
 501                 *
 502                 * we only support ethernet device type,
 503                 * which (according to RFC 1390) should
 504                 * always equal 1 (Ethernet).
 505                 */
 506
 507                arp->ar_hrd = htons(np->dev->type);
 508                arp->ar_pro = htons(ETH_P_IP);
 509                arp->ar_hln = np->dev->addr_len;
 510                arp->ar_pln = 4;
 511                arp->ar_op = htons(type);
 512
 513                arp_ptr = (unsigned char *)(arp + 1);
 514                memcpy(arp_ptr, np->dev->dev_addr, np->dev->addr_len);
 515                arp_ptr += np->dev->addr_len;
 516                memcpy(arp_ptr, &tip, 4);
 517                arp_ptr += 4;
 518                memcpy(arp_ptr, sha, np->dev->addr_len);
 519                arp_ptr += np->dev->addr_len;
 520                memcpy(arp_ptr, &sip, 4);
 521
 522                netpoll_send_skb(np, send_skb);
 523
 524                /* If there are several rx_hooks for the same address,
 525                   we're fine by sending a single reply */
 526                break;
 527        }
 528        spin_unlock_irqrestore(&npinfo->rx_lock, flags);
 529}
 530
 531int __netpoll_rx(struct sk_buff *skb)
 532{
 533        int proto, len, ulen;
 534        int hits = 0;
 535        const struct iphdr *iph;
 536        struct udphdr *uh;
 537        struct netpoll_info *npinfo = skb->dev->npinfo;
 538        struct netpoll *np, *tmp;
 539
 540        if (list_empty(&npinfo->rx_np))
 541                goto out;
 542
 543        if (skb->dev->type != ARPHRD_ETHER)
 544                goto out;
 545
 546        /* check if netpoll clients need ARP */
 547        if (skb->protocol == htons(ETH_P_ARP) &&
 548            atomic_read(&trapped)) {
 549                skb_queue_tail(&npinfo->arp_tx, skb);
 550                return 1;
 551        }
 552
 553        proto = ntohs(eth_hdr(skb)->h_proto);
 554        if (proto != ETH_P_IP)
 555                goto out;
 556        if (skb->pkt_type == PACKET_OTHERHOST)
 557                goto out;
 558        if (skb_shared(skb))
 559                goto out;
 560
 561        if (!pskb_may_pull(skb, sizeof(struct iphdr)))
 562                goto out;
 563        iph = (struct iphdr *)skb->data;
 564        if (iph->ihl < 5 || iph->version != 4)
 565                goto out;
 566        if (!pskb_may_pull(skb, iph->ihl*4))
 567                goto out;
 568        iph = (struct iphdr *)skb->data;
 569        if (ip_fast_csum((u8 *)iph, iph->ihl) != 0)
 570                goto out;
 571
 572        len = ntohs(iph->tot_len);
 573        if (skb->len < len || len < iph->ihl*4)
 574                goto out;
 575
 576        /*
 577         * Our transport medium may have padded the buffer out.
 578         * Now We trim to the true length of the frame.
 579         */
 580        if (pskb_trim_rcsum(skb, len))
 581                goto out;
 582
 583        iph = (struct iphdr *)skb->data;
 584        if (iph->protocol != IPPROTO_UDP)
 585                goto out;
 586
 587        len -= iph->ihl*4;
 588        uh = (struct udphdr *)(((char *)iph) + iph->ihl*4);
 589        ulen = ntohs(uh->len);
 590
 591        if (ulen != len)
 592                goto out;
 593        if (checksum_udp(skb, uh, ulen, iph->saddr, iph->daddr))
 594                goto out;
 595
 596        list_for_each_entry_safe(np, tmp, &npinfo->rx_np, rx) {
 597                if (np->local_ip && np->local_ip != iph->daddr)
 598                        continue;
 599                if (np->remote_ip && np->remote_ip != iph->saddr)
 600                        continue;
 601                if (np->local_port && np->local_port != ntohs(uh->dest))
 602                        continue;
 603
 604                np->rx_hook(np, ntohs(uh->source),
 605                               (char *)(uh+1),
 606                               ulen - sizeof(struct udphdr));
 607                hits++;
 608        }
 609
 610        if (!hits)
 611                goto out;
 612
 613        kfree_skb(skb);
 614        return 1;
 615
 616out:
 617        if (atomic_read(&trapped)) {
 618                kfree_skb(skb);
 619                return 1;
 620        }
 621
 622        return 0;
 
 
 ```
 623}
