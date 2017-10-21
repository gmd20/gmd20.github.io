##  好像比较互联网的请求的流控，比较经常提到的是google  guava 里面的ratelimit的实现。
这里可以看到一个完整的实现，应该是Token Bucket Filter上面稍微有些变化。
https://github.com/google/guava/blob/master/guava/src/com/google/common/util/concurrent/SmoothRateLimiter.java#L276:L309       
https://github.com/google/guava/blob/master/guava/src/com/google/common/util/concurrent/RateLimiter.java      


##  linux流量整形的实现
http://elixir.free-electrons.com/linux/latest/source/net/sched/sch_tbf.c   
http://elixir.free-electrons.com/linux/latest/source/net/netfilter/nft_limit.c   

我也是准备在内核实现一个类似网络流量控制。 但又不想tc这个这么复杂。linux的时间戳除了jiffies jiffies_64应该还可以用
ktime_get_ns()的
