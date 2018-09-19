```text
这几天在做一个网络包相关的bug，都是通过 Linux上的思科vpn客户端连到客户的服务器上面去测试的，抓包测试的时候，才发现wireshark没有办法把cisco vpn client接口的返回包抓下来，出去的包是可以抓的到的。bug终于有点头绪了，回来没事做到csdn逛了逛，想起这个问题来，就找了cisco的vpn client的代码，看看能不能发现是什么问题，

说Linux的网络的，发现两本书好像讲的不错

“Understanding_Linux_Network_Internals.chm” “?Prentice.Hall.The.Linux.Networking.Architecture.Design.and.Implementation.of.Network.Protocols.in.the.Linux.Kernel.chm”

反正我这种非专业人士读了，感觉对些网络概念的理解有很大帮助。

cisco的vpn client代码，网上很容易找到，我找到是这个 vpnclient-linux-x86_64-4.8.02.0030-k9-AMD64_ONLY_by_t3x.tar.gz，这个应该是没有打了新的2.6.35那些新版内核的补丁的，新版的内核net_device 结构变量，把那些hard_start_xmit函数整理到一个net_device_ops结构里面去了，所以在新的内核里面还要改一下。

先大概浏览vpn核模块的代码，关键的代码都是在?interceptor.c文件里面，之前也稍微看过一点点，有点了解。

像这种vpn的实现，就理解来说无非就是在网络包出口和入口的两个地方做点重新打包修改的什么，封装成IPSEC等协议吧，Cisco的这个代码还是很简单清晰的。

--------==出口的====-------

他替换了网络设备net_device 结构的hard_start_xmit (新版内核是net_device_ops->?ndo_start_xmit ) 函数，然后网络设备出去的包都会先调到他自己的挂载函数了，就可以在这里做包的修改和封装了。 因为自己也用了一些vpn头部，所以需要设置一下网络设备的什么mtu啦等。还有他替换了所有系统的以太网类型的设备，反正就是认为以太网类型的都是他可以用来做vpn的？？？ 不知道还有没有什么比直接替换挂钩更好的办法不？ 新版的ndo_start_xmit都是常量指针来的了，用这种办法还要强制转换替换。不过这种办法很简单容易理解就是了。不过我们之前发现这种实现办法和vmware的虚拟机虚拟网卡有冲突，当时我们是在vpn client里面直接跳过虚拟网卡来处理的。

    ndo_start_xmit 是一个很有意思的函数，比如可以自己搞个假的网络设备，然后调用真的网络设备的这个函数发送出去。中间可以自己做些修改什么的。就像cisco 这个vpn 客户端的内核模块做的一样。当然我也知道还有其他这样用的驱动^_^

----------------入口点---------------------

他这个实现很有意思，他代码是这样写的

?    /* find the handler for inbound IP packets by adding a dummy handler
     * for that packet type into the kernel. Because the packet handlers
     * are stored in a hash table, we'll be able to pull the original 
     * ip packet handler out of the list that dummy_pt was inserted into.*/
    kernel_memset(&dummy_pt, 0, sizeof(dummy_pt));
    dummy_pt.type = htons(ETH_P_IP);
    dummy_pt.func = recv_ip_packet_handler;

    dev_add_pack(&dummy_pt);
    /* this should be the original IP packet handler */
    default_pt = PACKET_TYPE_NEXT(&dummy_pt);
    /* there may be more than one other packet handler in our bucket,
     * so look through all the buckets */
    while (default_pt != NULL && default_pt->type != htons(ETH_P_IP))
    {
        default_pt = PACKET_TYPE_NEXT(default_pt);
    }
    if (!default_pt)
    {
        printk(KERN_DEBUG "No default handler found for %x protocol!!\n",
               dummy_pt.type);
        dev_remove_pack(&dummy_pt);
        error = VPNIFUP_FAILURE;
        goto error_exit;
    }
    /*remove the dummy handler handler */
    original_ip_handler.pt = default_pt;
    dev_remove_pack(&dummy_pt);

    /*and replace the original handler function with our function */
    original_ip_handler.orig_handler_func = original_ip_handler.pt->func;
    original_ip_handler.pt->func = recv_ip_packet_handler;

先通过?dev_add_pack 函数注册一个?ETH_P_IP协议的处理函数，得到一个链表指针，然后通过这个链表指针找到系统原有ip处理函数（这个应该是ip_rcv 函数?http://lxr.linux.no/linux+v2.6.36/net/ipv4/ip_input.c#L375）。为的就是找到原有函数，进行挂钩，这样就能拦截所有系统ip包入口点。这个也是很有趣的东东，之前我没有见过，刚刚发现?Understanding Linux Network Internals 一书的 “13.4. Protocol Handler Registration” 小节有说到?用?dev_add_pack 和 ?dev_remove_pack 函数来系统里面注册自己网络协议的办法，感兴趣的可以去看一下介绍，还有参考一下上面的例子。

我不知道他们这里怎么选择了这种办法，要是我的话，我可能考虑使用netfilter应该也是可以实现的，毕竟我感觉这种hook的办法可能不是很正式。

---------------------------------------------------------------------------------------

来到这里，问题应该比较清晰了，wireshark不能抓到回来的网络包，应该就是这个模块里面上传个上一级的地方少做了哪些步骤了。

我记得第一本书里面说了抓包是怎么实现的了，还有一个图，再自己看了一下,原文里面有一段是这么写的：


13.1. Overview of Network Stack

Network sniffers such as tcpdump and Ethereal are common users of AF_SOCKET sockets. You can see from the figure that

AF_PACKET sockets hand frames directly to dev_queue_xmit, and receive ingress frames directly from the network protocol

dispatcher routine (this latter point is addressed in Chapter 10).

结合书上所讲和内核源代码,Linux 的raw socket抓包实现应该是往系统注册一个PF_PACKET网络协议，然后直接把原始网络包上传给网络抓包程序。raw socket的实现是在http://lxr.linux.no/linux+v2.6.36/net/packet/af_packet.c 里面做的吧。

系统调用各个协议来处理网络包应该是这个地方

http://lxr.linux.no/linux+v2.6.36/net/core/dev.c#L2817
static int __netif_receive_skb(struct sk_buff *skb)

2878        list_for_each_entry_rcu(ptype, &ptype_all, list) {
2879                if (ptype->dev == null_or_orig || ptype->dev == skb->dev ||
2880                    ptype->dev == orig_dev) {
2881                        if (pt_prev)
2882                                ret = deliver_skb(skb, pt_prev, orig_dev); //调用各个协议的处理函数
2883                        pt_prev = ptype;
2884                }
2885        }

2614static inline int deliver_skb(struct sk_buff *skb,
2615                              struct packet_type *pt_prev,
2616                              struct net_device *orig_dev)
2617{
2618        atomic_inc(&skb->users);
2619        return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
2620}
2621

而注册的抓包协议PF_PACKET的处理函数应该是这个
static int packet_rcv_spkt(struct sk_buff *skb, struct net_device *dev,
                           struct packet_type *pt, struct net_device *orig_dev)

是在这个地方注册进去的
http://lxr.linux.no/linux+v2.6.36/net/packet/af_packet.c#L1493
/*
1447 *      Create a packet of type SOCK_PACKET.
1448 */
1449
1450static int packet_create(struct net *net, struct socket *sock, int protocol,
1451                         int kern)


1477        po = pkt_sk(sk);
1478        sk->sk_family = PF_PACKET;
1479        po->num = proto;


   if (sock->type == SOCK_PACKET)
1493                po->prot_hook.func = packet_rcv_spkt;
1494
1495        po->prot_hook.af_packet_priv = sk;
1496
1497        if (proto) {
1498                po->prot_hook.type = proto;
1499                dev_add_pack(&po->prot_hook);   注册协议
1500                sock_hold(sk);
1501                po->running = 1;
1502        }

由此就可以大概猜测得出，cisco vpn抓包没能捕获回来的包，应该就是解码的时候?packet_rcv_spkt 函数能能认出解码之前的包，直接丢弃了？？应该是解码后，在传个这个函数处理的吧。我们再来看一下cisco自己搞的ip 协议处理函数。

#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,14)
static int
recv_ip_packet_handler(struct sk_buff *skb,
                       struct net_device *dev, 
                       struct packet_type *type,
                       struct net_device *orig_dev)
#else
static int
recv_ip_packet_handler(struct sk_buff *skb,
                       struct net_device *dev, 
                       struct packet_type *type)
#endif
{
    int rc2 = 0;
    int tmp_rc = 0;
    CNISTATUS rc = 0;
    CNIPACKET NewPacket = NULL;
    CNIFRAGMENT Fragment = NULL;
    CNIFRAGMENT MacHdr = NULL;
    PVOID lpReceiveContext = NULL;
    ULONG ulFinalPacketSize;
    BINDING *pBinding = NULL;
    struct ethhdr ppp_dummy_buf;
    int hard_header_len;

#ifdef MOD_INC_AND_DEC
    MOD_INC_USE_COUNT;
#endif
    if (dev->type == ARPHRD_LOOPBACK)
    {
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,14)
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type, dev);
#else
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type);
#endif
        goto exit_gracefully;
    }

    /* Don't handle non-eth non-ppp packets */
    if (!supported_device(dev))
    {
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,14)
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type, dev);
#else
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type);
#endif
        goto exit_gracefully;
    }

    pBinding = getbindingbydev(dev);

    /* if we don't have a binding, this is a new device that
     * has been brought up while the tunnel is up. For now,
     * just pass the packet
     */
    if (!pBinding)
    {
        static int firsttime = 1;
        if (firsttime)
        {
            printk(KERN_DEBUG "RECV: new dev %s detected\n", dev->name);
            firsttime = 0;
        }

#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,14)
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type, dev);
#else
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type);
#endif
        goto exit_gracefully;
    }

    //only need to handle IP packets.
    if (skb->protocol != htons(ETH_P_IP))
    {
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,14)
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type, dev);
#else
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type);
#endif
        goto exit_gracefully;
    }

#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,19)
    if (skb->ip_summed == CHECKSUM_PARTIAL)
#else
    if (skb->ip_summed == CHECKSUM_HW)
#endif
    {
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,7)
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,10)
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,19)
       if (skb_checksum_help(skb))
#else
       if (skb_checksum_help(skb,1))
#endif
#else
       if (skb_checksum_help(&skb,1))
#endif // LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,10)
       {
           dev_kfree_skb(skb);
           skb = NULL;
           goto exit_gracefully;
       }
#else
       skb->ip_summed = CHECKSUM_NONE;
#endif
    }

    reset_inject_status(&pBinding->recv_stat);
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,22)
    if (skb->mac_header)
#else
    if (skb->mac.raw)
#endif
    {
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,22)
        hard_header_len = skb->data - skb->mac_header;
#else
        hard_header_len = skb->data - skb->mac.raw;
#endif
        if ((hard_header_len < 0) || (hard_header_len > skb_headroom(skb)))
        {
            printk(KERN_DEBUG "bad hh len %d\n", hard_header_len);
            hard_header_len = 0;
        }
    }
    else
    {
        hard_header_len = 0;
    }

    pBinding->recv_real_hh_len = hard_header_len;

    switch (hard_header_len)
    {
    case ETH_HLEN:
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,22)
        CniNewFragment(ETH_HLEN, skb->mac_header, &MacHdr, CNI_USE_BUFFER);
#else
        CniNewFragment(ETH_HLEN, skb->mac.raw, &MacHdr, CNI_USE_BUFFER);
#endif
        break;
    case IPPP_MAX_HEADER:
    case 0:
        MacHdr = build_ppp_fake_mac_frag(&ppp_dummy_buf);
        break;
    default:
        printk(KERN_DEBUG "unknown mac header length (%d)\n", hard_header_len);
        dev_kfree_skb(skb);
        skb = NULL;
        goto exit_gracefully;
    }

    CniNewFragment(skb->len, skb->data, &Fragment, CNI_USE_BUFFER);
    ulFinalPacketSize = skb->len;

    rc = CNICallbackTable.Receive((void *) &pBinding,
                                  &lpReceiveContext,
                                  &MacHdr,
                                  &Fragment, &NewPacket, &ulFinalPacketSize);

    switch (rc)
    {
    case CNI_CONSUME:
        tmp_rc = CNICallbackTable.ReceiveComplete(pBinding,
                                                  lpReceiveContext, NewPacket);

        dev_kfree_skb(skb);
        rx_packets++;
        if (pBinding->recv_stat.called)
        {
            rc2 = pBinding->recv_stat.rc;
        }
        break;
    case CNI_CHAIN:
        tmp_rc = CNICallbackTable.ReceiveComplete(pBinding,
                                                  lpReceiveContext, NewPacket);

#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,14)
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type, dev);   //调用ip协议处理函数
#else
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type);
#endif

        if (pBinding->recv_stat.called)
        {
            rc2 = pBinding->recv_stat.rc;
        }

        break;
    case CNI_DISCARD:
        dev_kfree_skb(skb);
        rx_dropped++;
        break;
    default:
        printk(KERN_DEBUG "RECV: Unhandled case in %s rc was %x\n",
               __FUNCTION__, (uint) rc);

        dev_kfree_skb(skb);
        rx_dropped++;
    }
exit_gracefully:
    if (MacHdr)
    {
        CniReleaseFragment(MacHdr, CNI_KEEP_BUFFERS);
    }
    if (Fragment)
    {
        CniReleaseFragment(Fragment, CNI_KEEP_BUFFERS);
    }
#ifdef MOD_INC_AND_DEC
    MOD_DEC_USE_COUNT;
#endif

    return rc2;
}


可以看到他解码之后，是调用了?系统原有的IP处理函数的（ip_rcv）我看到，这个ip_rcv 也是调用了NF_HOOK函数的，所有那些netfilter的iptables那些过滤设施还能正常工作。而我们要解决抓包不成功的问题，只要模拟他的办法，用dev_add_pack dev_remove_pack 得到PF_PACKET的原有函数地址，然后最后再调用多一次处理这个解码以后的包应该就可以了。至此问题解决，明天去到公司再测试一下。



中午按照上昨天的理解，修改了一下cisco的vpn 客户端的内核模块，不过不能正常工作。有必要再深入看一下?PF_PACKET这个抓包接口在Linux上面的实现。首先我昨天理解错误的一个地方 是：?PF_PACKET不是protocol来的，应该是属于family来的，

按照流程看一下代码吧：

（1）用户创建一个raw socket来抓包是这样的

?mysocket= socket(intsocket_family, int socket_type, int protocol);

packet_socket = socket(PF_PACKET, 抓包对应的 有??SOCK_PACKET SOCK_RAW SOCK_DGRAM ??，协议有 ?ETH_P_ALL表示所有的类型的包都会被抓到，?ETH_P_802_3/ETH_P_IP表示ip包等）。

简单的说这第3个参数才是对应内核注册函数void dev_add_pack(struct packet_type *pt) 所使用到的pack_typeJ结构的protocol属性来的。

另外IP层也还有一个raw socket，在?http://lxr.linux.no/linux+v2.6.36/net/ipv4/raw.c 里面实现，那个应该是用

socket(PF_INET，。。。。 来创建的。 不过wireshark应该是使用?socket(PF_PACKET的办法，不然不会抓不到包，就不知道他第 3个参数protocol传的是什么。

（2） 这个socket(PF_PACKET 是如何和内核的dev_add_pack 关联起来的。

socket系统调用 

调用到http://lxr.linux.no/linux+v2.6.36/net/socket.c 里面的?sock_create 函数，在?sock_create 函数里面又会根据用户的family参数也就是PF_PACKET 调用

?http://lxr.linux.no/linux+v2.6.36/net/packet/af_packet.c#L1445 里面

static int packet_create(struct net *net, struct socket *sock, int protocol,
1451                         int kern)
函数，packet_create 函数里面就会根据相应传过来的protocol使用dev_add_pack 来注册相应的协议处理函数。

为什么sock_create 会找到packet_create 来呢？因为在文件

?http://lxr.linux.no/linux+v2.6.36/net/packet/af_packet.c

里面有

?2576static const struct proto_opspacket_ops = {
2577        .family =       PF_PACKET,
2578        .owner =        THIS_MODULE,
2579        .release =      packet_release,
2580        .bind =         packet_bind,
2581        .connect =      sock_no_connect,
2582        .socketpair =   sock_no_socketpair,
2583        .accept =       sock_no_accept,
2584        .getname =      packet_getname,
2585        .poll =         packet_poll,
2586        .ioctl =        packet_ioctl,
2587        .listen =       sock_no_listen,
2588        .shutdown =     sock_no_shutdown,
2589        .setsockopt =   packet_setsockopt,
2590        .getsockopt =   packet_getsockopt,
2591        .sendmsg =      packet_sendmsg,
2592        .recvmsg =      packet_recvmsg,
2593        .mmap =         packet_mmap,
2594        .sendpage =     sock_no_sendpage,
2595};

2597static const struct net_proto_familypacket_family_ops = {
2598        .family =       PF_PACKET,
2599        .create =       packet_create,
2600        .owner =       THIS_MODULE,
2601};
2602
2707static int __initpacket_init(void)
2708{
2709 int rc = proto_register(&packet_proto, 0);
2710
2711        if (rc != 0)
2712                goto out;
2713
2714        sock_register(&packet_family_ops);
2715        register_pernet_subsys(&packet_net_ops);
2716        register_netdevice_notifier(&packet_netdev_notifier);
2717out:
2718        return rc;
2719}
2720
2721module_init(packet_init);

这写socket类的函数， 向系统注册了相应的family的处理函数了。

（3) Linux系统里面，网络设备接受到和发送出去的包是如何转给相应的socket的。

收包时，都会经过?int netif_receive_skb(struct sk_buff *skb)
这个函数 ，这个函数在http://lxr.linux.no/linux+v2.6.36/net/core/dev.c 里面

这个调用到

?static int __netif_receive_skb(struct sk_buff *skb)

2918        type = skb->protocol;
2919        list_for_each_entry_rcu(ptype,
2920                        &ptype_base[ntohs(type) & PTYPE_HASH_MASK], list) {
2921                if (ptype->type == type && (ptype->dev == null_or_orig ||
2922                     ptype->dev == skb->dev || ptype->dev == orig_dev ||
2923                     ptype->dev == orig_or_bond)) {
2924                        if (pt_prev)
2925                                ret = deliver_skb(skb, pt_prev, orig_dev); 
2926                        pt_prev = ptype;
2927                }
2928        }

发包的时候，都会经过?dev_queue_xmit 这个函数

这里就通过 deliver_skb 调用了所有系统注册的网络协议的处理函数了。这个下面是这样的

1938int dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev,
1939                        struct netdev_queue *txq)
1940{
1941        const struct net_device_ops *ops = dev->netdev_ops;
1942        int rc = NETDEV_TX_OK;
1943
1944        if (likely(!skb->next)) {
1945                if (!list_empty(&ptype_all))
1946                        dev_queue_xmit_nit(skb, dev); //在这恶函数里面会回发给所有网络协议注册函数精进行处理，就是通过这个传回给PF_PACKET注册的网络协议的packet_rcv 函数了
1947
1948                /*
1949                 * If device doesnt need skb->dst, release it right now while
1950                 * its hot in this cpu cache
1951                 */
1952                if (dev->priv_flags & IFF_XMIT_DST_RELEASE)
1953                        skb_dst_drop(skb);
1
1980                rc = ops->ndo_start_xmit(skb, dev);   //调用网卡驱动发送函数出去
1981                if (rc == NETDEV_TX_OK)
1982                        txq_trans_update(txq);
1983                return rc;


1984        }


1500/*
1501 *      Support routine. Sends outgoing frames to any network
1502 *      taps currently in use.
1503 */
1504
1505static void dev_queue_xmit_nit(struct sk_buff *skb, struct net_device *dev)

自己注册的PF_PACKET对应的各个网络协议的对应出来函数，应该就是?

?static int packet_rcv(struct sk_buff *skb, struct net_device *dev,
533                      struct packet_type *pt, struct net_device *orig_dev)

static int packet_rcv_spkt(struct sk_buff *skb, struct net_device *dev,
340                           struct packet_type *pt, struct net_device *orig_dev)

两个，这两个里面就会传给相应的socket了。

现 在再来看看cisco vpn的问题，他代码里面如上文所说，只是替换了一个?ETH_P_IP 协议的处理函数，但wireshark等程序也可能是注册同样的ETH_P_IP协议的处理函数的，或者?ETH_P_ALL等。 现在的问题就是解码后的skb，在现有代码里面只是调用了一个原有的ETH_P_IP协议函数来进行再次处理。

解决办法，再像他这样hook 原有ETH_P_IP 协议的处理函数的办法应该是不行了，因为这个可能有很多个都要处理啊。一个办法就是重新排队这个skb，然后自动重走整个流程，但可能要做一下mark，不然在cisco vpn就会重复处理这个skb了。

还有在内核代码里面看到这样一个函数 ，可惜不是导出的

2695/*
2696 *      netif_nit_deliver - deliver received packets to network taps
2697 *      @skb: buffer
2698 *
2699 *      This function is used to deliver incoming packets to network
2700 *      taps. It should be used when the normal netif_receive_skb path
2701 *      is bypassed, for example because of VLAN acceleration.
2702 */
2703void netif_nit_deliver(struct sk_buff *skb)
2704{
2705        struct packet_type *ptype;
2706
2707        if (list_empty(&ptype_all))
2708                return;
2709
2710        skb_reset_network_header(skb);
2711        skb_reset_transport_header(skb);
2712        skb->mac_len = skb->network_header - skb->mac_header;
2713
2714        rcu_read_lock();
2715        list_for_each_entry_rcu(ptype, &ptype_all, list) {
2716                if (!ptype->dev || ptype->dev == skb->dev)
2717                        deliver_skb(skb, ptype, skb->dev);
2718        }
2719        rcu_read_unlock();
2720}

应该可以参考一下，重新requeue一下这个解码后的skb，应该就可以让wireshark给捕获到了。需要调试一下。




===========================================================12月6号补充
------------------用kprobe 挂钩 dev_add_pack函数 --------------------
/* Proxy routine having the same arguments as actual do_fork() routine */
void jdo_dev_add_pack(struct packet_type *pt)
{
       // printk(KERN_INFO "jprobe: clone_flags = 0x%lx, stack_size = 0x%lx,"
       //                 " regs = 0x%p\n",
       //        clone_flags, stack_size, regs);

          
       printk ("在设备上 %s 上创建捕获接口，类型 =%d ，处理函数地址= %p\n",pt->dev->name, ntohs( pt->type),pt->func);

        /* Always end with a call to jprobe_return(). */
        jprobe_return();

}

可以得到以下输出

------------------------------
[ 6967.330146] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6967.348045] 在设备上 lo 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6967.356520] 在设备上 lo 上创建捕获接口，类型 =3 ，处理函数地址= c05a3550
[ 6967.372573] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6967.388519] 在设备上 eth0 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6967.396529] 在设备上 eth0 上创建捕获接口，类型 =3 ，处理函数地址= c05a3550
[ 6967.416147] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6967.432028] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a3550
[ 6967.448222] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6967.464020] 在设备上 lo 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6967.476020] 在设备上 lo 上创建捕获接口，类型 =3 ，处理函数地址= c05a3550
[ 6967.492056] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6967.504011] 在设备上 eth0 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6967.516011] 在设备上 eth0 上创建捕获接口，类型 =3 ，处理函数地址= c05a3550
[ 6967.532073] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6967.548020] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a3550
[ 6978.319957] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6978.332015] 在设备上 lo 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6978.340018] 在设备上 lo 上创建捕获接口，类型 =3 ，处理函数地址= c05a3550
[ 6978.360054] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6978.376008] 在设备上 eth0 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6978.384025] 在设备上 eth0 上创建捕获接口，类型 =3 ，处理函数地址= c05a3550
[ 6978.400135] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6978.416018] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a3550
[ 6978.436517] 在设备上 (null) 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6978.448521] 在设备上 eth0 上创建捕获接口，类型 =3 ，处理函数地址= c05a2480
[ 6978.448553] device eth0 entered promiscuous mode
[ 6978.464518] 在设备上 eth0 上创建捕获接口，类型 =3 ，处理函数地址= c05a3550
-----------------------------------


#define ETH_P_ALL       0x0003          /* Every packet (be careful!!!) */

可以知道wireshark创建的是所有协议的，抓包的应该都是用这个

widebright@gdeng:~/桌面/kprobe$ cat /proc/kallsyms |grep packet_rcv
c05a2480 t packet_rcv
c05a3550 t tpacket_rcv      
c05a39f0 t packet_rcv_spkt    

可以知道，他注册的处理函数是这几个。

====================================================
关于具体的代码修改，
前 面说了，他的vpn的代码实现的问题，主要是他直接替换了人家系统ip协议的处理函数ip_rcv 这个，在解码后不再给所有的的协议分发，而是直接调用原来的ip_rcv处理解码后的包，所以其他的抓包协议ETH_P_ALL对应的的抓包函数就没有机 会在处理解码出来的包了。

修改的办法有两个：
（1)
有一个办法是，在调用ip_rcv之前，先调用前面所说的 netif_nit_deliver函数，netif_nit_deliver里面就给所有ptype_all链表下面的协议处理函数分发了，看一下 dev_add_pack函数就可以知道所有ETH_P_ALL类型的都是加到ptype_all这个链表里面去的。vlan处理函数有一个使用的例子， 他也是绕过了先前在——__netif_receive_skb里面分发的地方，所有后面也要调用一下netif_nit_deliver来通知所有的抓 包接口。
    因为netif_nit_deliver函数不是导出函数，所以，可以在运行时用下面这个函数来找到netif_nit_deliver函数的地址，在强制转换成函数指针调用。

linux/include/linux/kallsyms.h
171/* Lookup the address for this symbol. Returns 0 if not found. */
172unsigned long kallsyms_lookup_name(const char *name)
173{
174        char namebuf[KSYM_NAME_LEN];
175        unsigned long i;
176        unsigned int off;
177
178        for (i = 0, off = 0; i < kallsyms_num_syms; i++) {
179                off = kallsyms_expand_symbol(off, namebuf);
180
181                if (strcmp(namebuf, name) == 0)
182                        return kallsyms_addresses[i];
183        }
184        return module_kallsyms_lookup_name(name);
185}
186EXPORT_SYMBOL_GPL(kallsyms_lookup_name);

kallsyms_lookup_name这个函数很有意思阿，有了它就可以找到内核里面各个函数的地址了，然后自己有调用了，好像我以前还是写函数自己在/proc/kallsyms
文件里面去搜索的，这个kallsyms_lookup_name函数应该是后来才有的吧。
   我没有测试国这个办法，不过理论是应该是可行的^_^



(2)
   第二种办法，是让修改后的skb，重新回头再重新分发一下，代码是这样的，就是把解码后的skb当作重新进来的包，调用一下 netif_rx，netifrx就会加到队列里面重新处理了。等会skb就会通过 netif_receive_skb函数，在__netif_receive_skb函数里面调用各个协议的分发函数，抓到接口就会得到了。

---------------------------------------------------------
         skb->mark = 1007;
         skb_orphan(skb);
         nf_reset(skb);
         skb->tstamp.tv64 = 0;
         skb->pkt_type = PACKET_HOST;
         pBinding->recv_stat.rc = netif_rx(skb);

--------------------------------------------------------

用上面那样的代码替换cisco vpn里面直接调用 ip_rcv的地方，就可以了。注意一共要改两个地方，其中一个是在interceptor.c 文件的recv_ip_packet_handler函数里面的这个地方
“rc2 = original_ip_handler.orig_handler_func(skb, dev, type, dev);“
另外一个是 linuxcniapi.c 文件CniInjectReceive函数里面的
“pBinding->recv_stat.rc = pBinding->InjectReceive(skb,

这个地方。因为重新分发后，包有会重新进来ip_rcv这个函数里面，所以在recv_ip_packet_handler的前面跳过我们做过标记的skb，不用再解码了。
这样写


    ////////////////////////
    //我们解码过的包重新被分发到这里来了。
    if( skb->mark == 1007 )
    {
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,14)
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type, dev);
#else
        rc2 = original_ip_handler.orig_handler_func(skb, dev, type);   //直接调用ip_rcv函数处理包
#endif
        goto exit_gracefully;
    }
    ////////////////////////





这样，重现编译vpn的内核模块，再用wireshark抓包就可以抓到了。
不 过抓到的包是人不出来的，只看到是说以太网的包，具体的协议人不出来，不过如果你选 wireshark的抓包设备为 “any”的时候，包是可以正确识别出来的。通过调试发现，抓包的协议的函数在packet_rcv_spkt里面会尝试在skb->data的前 面在skb->push一下以得到 以太网头部，包括mac那些。但是cisco的vpn里面解码后的包，他是不把ethdr信息给填进去的，他搞了个0字节的以太网头，所有 wireshark解码的时候就得不到这以太网头部，认不出这个包来。


解码时候，recv_ip_packet_handler 函数里面在调用之前，skb->dev 对应的设备还是eth0，mac头部长度还是14，这个MacHdr也是把这个mac信息传进去的，

    CniNewFragment(skb->len, skb->data, &Fragment, CNI_USE_BUFFER);
    ulFinalPacketSize = skb->len;

    rc = CNICallbackTable.Receive((void *) &pBinding,
                                  &lpReceiveContext,
                                  &MacHdr,
                                  &Fragment, &NewPacket, &ulFinalPacketSize);

但他调用了CNICallbackTable.Receive ，再转发到

NOREGPARM CNISTATUS
CniInjectReceive(IN CNIBINDING Binding,
                 IN PVOID ReceiveContext,
                 IN CNIFRAGMENT MacHeader, IN CNIPACKET Packet,
                 IN ULONG ulSize)

的时候，Binding是被改成他的vpn的虚拟设备cipsec0了，这时MacHeader 已经不会把mac给传过来了，cipsec0设备对应的pBinding->recv_real_hh_len
值 也是被设置为0了。 他内部转发的时候，他并不伪造一个假的 以太网头部出来。可能他们也是为了简化吧，正像他们直接考虑替换ip_rcv函数一样。为了packet_rcv_spkt函数能够处理，也是可以在后面 在伪造一个eth头部的，然后再调用netif_rx的。不过cisco的人这不该cni解码部分代码并没有给出来，都是用object提供的，最好是再 他里forward时候就给指定了才好吧，不过自己随便搞个ip包类型，搞两个mac地址进去应该可以的。我就没有继续试了。

wireshark 里面用any 设备来抓包时，它采用的是 SOCK_DGRAM这个参数，抓到的是 “Linux cooked capture“格式，他是忽略以太网头部的，所以能够正确解码协议出来。应该对应的网络协议处理函数是packet_rcv，可以去看一下是不是忽略掉 了以太网头部了。

网上可以的看到“Linux cooked capture“格式 的解释
//---------------------------------------------------------------------------
ETHEREAL-USERS: DECEMBER 2004
Subject: Re: [Ethereal-users] What is "Linux cooked capture" and why does it add 2 bytes?
From: Guy Harris <gharris@xxxxxxxxx>
Date: Tue, 28 Dec 2004 11:49:45 -0800
Rutger Thomschitz wrote:
However, just
with VTun (an
"Linux cooked
"Linux cooked
recently, as I was doing an experiment
opensource VPN solution) I noticed Ethereal reporting
capture", which seems to add an additonal 2 bytes. What is
capture"?
On Linux, packet capturing is done by opening a socket. In systems with a 2.2 or later kernel, the socket is a
PF_PACKET socket, either of type SOCK_RAW or SOCK_DGRAM.
A SOCK_RAW socket supplies the packet data including what the driver specified, when constructing the socket
buffer (skbuff) holding the packet, to be the packet's link-layer header; a SOCK_DGRAM packet supplies only data
above what was specified by the driver to be the link-layer header.
For the purposes of libpcap, which is the library used by programs such as tcpdump, Ethereal/Tethereal, snort,
etc. to capture network traffic, a SOCK_RAW socket is usually the appropriate type of socket on which to
capture, and is what's used.
Unfortunately, the purported link-layer header might be missing (as is the case for some PPP interfaces), or
might contain random unpredictable amounts of data (as is the case for at least some interfaces using ISDN), or
might not contain enough data to determine the type of the packet (as is the case with at least some ATM
interfaces), so capturing with a SOCK_RAW socket doesn't always work well.
For interfaces of those types - and for interfaces of a type that libpcap currently doesn't have code to support
- libpcap uses a SOCK_DGRAM socket, and constructs a fake link-layer header from the address supplied by a
"recvfrom()" on that socket.
A "Linux cooked capture" is one done with libpcap using a SOCK_DGRAM socket.

//-----------------------------------------------------




总 结：   修改后，抓包就用wireshark的any device抓包即可，然后过滤ip和协议等就可以了。我重新排队的办法，应该会稍微影响一下性能，不过暂时感觉不出来。通过这个比较熟悉网络包的处理的 过程了，比如进来的入口netif_receive_skb   出去的包的通过dev_hard_start_xmit都在内核的/net/core/dev.c文件里面。还有dev_forward_skb 函数可以把一个包转到另外的设备等。 skb->dev 就是对应的netdevice，skb->mark用来标记包的不同，用于不同的处理等等。skb的重新排队等等。


另外kprobe的使用真是非常的方便的，可以在任意的内核函数调用到的地方打印信息，检查参数的值等。


```
