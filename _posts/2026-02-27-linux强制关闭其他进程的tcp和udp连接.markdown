最近遇到 golang进程变僵尸，但父进程是 init pid=1 的，监听的 tcp 和 udp在进程变僵尸后 init 进程没有回收（busybox的 init可能有些问题？）
ss 命令 这个golang 监听的连接和端口还被占用着，init进程又不能结束掉，这个资源一直没法是否

之前发现 ss命令 有一个已经建立的的tcp连接的， 但监听socket估计也关不了
```bash
  ss -K dst  127.0.0.1:9501
```

问了几个AI都每找到方法，只有deepseek提供了一个思路，看来deepseek还是厉害啊。 内核5.19+引入的 BPF bpf_sock_destroy kfunc
```c
// BPF程序段，附加到udp迭代器
SEC("iter/udp")
int destroy_udp_socket(struct bpf_iter__udp *ctx)
{
    struct sock *sk = ctx->udp_sk; // 获取当前迭代到的socket
    if (!sk)
        return 0;

    // 检查socket的本地地址和端口是否为 127.0.0.1:5353
    // 这里需要读取sk->__sk_common中的字段，如 skc_num (端口), skc_rcv_saddr (接收地址)
    // 这部分代码需要根据内核结构体定义来写，这里仅为逻辑示意
    if (sk->__sk_common.skc_num == 5353 && 
        sk->__sk_common.skc_rcv_saddr == ipv4_addr(127,0,0,1)) {
        
        // 调用kfunc销毁socket
        bpf_sock_destroy(sk);
    }

    return 0;
}
```

网上确实有 bpf_sock_destroy 介绍，结合 “BPF Iterators ” 不知道能找到对应的socket然后关闭不。
"如何终结已存在的TCP连接?——eBPF篇" [https://segmentfault.com/a/1190000044446907]

[https://docs.kernel.org/bpf/bpf_iterators.html#how-to-use-bpf-iterators]

