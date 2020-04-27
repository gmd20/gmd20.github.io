https://groups.google.com/forum/#!topic/bbr-dev/4jL4ropdOV8
https://patchwork.ozlabs.org/project/netdev/patch/20170516112436.10189-1-edumazet@google.com/
https://elixir.bootlin.com/linux/latest/source/net/ipv4/tcp_output.c#L2298

TCP的BBR算法是要求tcp pacing的，老的内核tcp pacing是在sch_fq.c 里面实现的， 所以只有启用tc qdisc fq后BBR才能工作正常，
Redhat 8的文档也还是这么写的。
但后来google的Eric Dumazet又改了一下在tcp 层实现了一个tcp pacing，如果fq 没有开的话，会自动failover到tcp层的pacing机制，
所以内核4.13版本之后不启用fq应该bbr也是能正常工作的了。核对了一下centos 8的代码，应该是包含了这个patch的了。
