Receive Side Scaling (RSS) with eBPF and CPUMAP
https://developers.redhat.com/blog/2021/05/13/receive-side-scaling-rss-with-ebpf-and-cpumap#faster_software_receive_steering_with_xdp
https://github.com/torvalds/linux/blob/master/kernel/bpf/cpumap.c

就是XDP的BPF里面的XDP_REDIRECT动作，增加一个 CPUMAP的重定向，就是使用bpf_redirect_map 这个函数吧，现在redirect有两个动作了：
bpf_redirect(ifindex, flags)  转发到ifindex对应的网卡出去
bpf_redirect_map(bpf_map, index_key, flags)  转发到index_key 对应cpu进行处理。

按照文章的说法，这个redirect发生在分配skb内存之前，后续的skb内存分配和包处理都是在同一个cpu上，对cpu缓存的利用更有效率了。作为对比RFS应该是分配完skb
后才通过cpu中断队列转发到别的cpu的。
之前测试RFS确实没看到什么明显的提升，不知道这个xdp cpumap实现的rss效果怎么样了。
这里有人利用这个配合tc mq多队列，解决单队列的全局锁的问题。
https://github.com/rchac/LibreQoS  

反正这个xdp能用上的场景确实会比较高效吧，比如防火墙防ddos从源头都拦截了恶意包，避免后续是skb分配的，确实是最高效率的。
但xdp感觉不太适合复杂的应用，只适合固定做些固定的简单的东西。比如这cpu map，估计可以做基于ip的负载均衡，但做不了nat 原始ip的负载均衡了吧。
