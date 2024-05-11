我一直以为linux内核的conntrack是默认启动的，昨天才发现不是的。经过反复测试阅读源码才知道是“按需”启动的。

通过反复测试和核对系统的 连接跟踪表，发现只有“ iptables -t nat -A  POSTROUTING ” 创建了snat规则，这个ct才开始运行的。
```text
cat /proc/sys/net/netfilter/nf_conntrack_count
conntrack -L -f ipv6
conntrack -L
kprobe 给ipv4_conntrack_in 函数设置勾子打印日志，（bpftrace 也可以吧）
````
然后找了半天，/proc  目录下也没有对应的 开关设置 （看源码好像6.0内核有开关了）。

最后找到这两个函数：
```text
/* load module; enable/disable conntrack in this namespace */
int nf_ct_netns_get(struct net *net, u8 nfproto);          有需要的模块可以调用这个两个函数开 开关 conntrack机制。
void nf_ct_netns_put(struct net *net, u8 nfproto);
```

 就是nft_ct这些模块，根据需要才调用  nf_ct_netns_get 函数启动 内核里面 conntrack机制， 不然内核默认是没有做连接跟踪的。
 大概流程是这样：
 
```text
nft_ct_get_init   // nft ct 初始化，才通知系统 需要启动  ct 机制。
  nf_ct_netns_get
    nf_ct_netns_inet_get     这个才注册 ipv4_conntrack_ops netfilter回调钩子
      nf_ct_netns_do_get


      ipv4_conntrack_in   连接跟踪的入口 
         nf_conntrack_in
            resolve_normal_ct
               nf_ct_get_tuple 抽取连接的标记信息，比如ip端口等信息，特殊协议字段等
                 __nf_conntrack_find_get
                 init_conntrack   为新连接创建对应的nf_conn ct

```

nf_conntrack  有两个依赖，nf_defrag_ipv4 和 nf_defrag_ipv6  ， 这两个钩子 还在 ipv4_conntrack_in 函数之前运行的， 是把 分片包组装起来的吧、

如果 不需要连接跟踪的，可以“nf_ct_set(skb, NULL, IP_CT_UNTRACKED); ”跟skb标记成IP_CT_UNTRACKED 的。
 


这个文章写的挺好，一开始搜索到了，但没细看，里面写的还是挺清楚的。
Connection tracking (conntrack) - Part 1: Modules and Hooks   
https://thermalcircle.de/doku.php?id=blog:linux:connection_tracking_1_modules_and_hooks
