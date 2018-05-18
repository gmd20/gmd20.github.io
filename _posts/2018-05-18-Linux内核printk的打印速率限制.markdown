如果在一些关键的路径上，比如网络包处理路径，大量用printk打印东西，是会拖垮系统的。   
要对这种关键路径上可能频繁打印的地方，做一些频率限制，可以避免估计触发打印的DDOS攻击等。


这个新版的内核已经提供了printk_ratelimited，  pr_err_ratelimited ， pr_warn_ratelimited 等宏的定义，直接使用就行了。
```c
#define printk_ratelimited(fmt, ...)					\
({									\
	static DEFINE_RATELIMIT_STATE(_rs,				\
				      DEFAULT_RATELIMIT_INTERVAL,	\
				      DEFAULT_RATELIMIT_BURST);		\
									\
	if (__ratelimit(&_rs))						\
		printk(fmt, ##__VA_ARGS__);				\
})

#define pr_err_ratelimited(fmt, ...)					\
	printk_ratelimited(KERN_ERR pr_fmt(fmt), ##__VA_ARGS__)
#define pr_warn_ratelimited(fmt, ...)					\
	printk_ratelimited(KERN_WARNING pr_fmt(fmt), ##__VA_ARGS__)
  
  
 #define WARN_RATELIMIT(condition, format, ...)			\
({								\
	static DEFINE_RATELIMIT_STATE(_rs,			\
				      DEFAULT_RATELIMIT_INTERVAL,	\
				      DEFAULT_RATELIMIT_BURST);	\
	int rtn = !!(condition);				\
								\
	if (unlikely(rtn && __ratelimit(&_rs)))			\
		WARN(rtn, format, ##__VA_ARGS__);		\
								\
	rtn;							\
})

```

这个依赖https://elixir.bootlin.com/linux/latest/source/include/linux/ratelimit.h 这个接口，   
不单单printk，自己其他地方有需要自定义的限速，应该都可以通过___ratelimit来实现的。   
看实现每个printk_ratelimited  都有一个自己独立的速率状态了。


旧版本的2.6的内核还没有这个___ratelimit， 只有printk_ratelimit和net_ratelimit，这个很多printk语句共享一个全局的锁了。
用法也没有新版内核这么方便。旧版本的是这么写的
```c
int net_msg_cost = 5*HZ;
int net_msg_burst = 10;

/* 
 * All net warning printk()s should be guarded by this function.
 */ 
int net_ratelimit(void)
{
	return __printk_ratelimit(net_msg_cost, net_msg_burst);
}


if (net_ratelimit())
    pr_err("Not enough memory to allocate rx buffer\n");
    
if (printk_ratelimit())
    printk("dma_sync_single_for_device: unsupported dir %u\n", dir);    
```


```c
/*
 * printk rate limiting, lifted from the networking subsystem.
 *
 * This enforces a rate limit: not more than one kernel message
 * every printk_ratelimit_jiffies to make a denial-of-service
 * attack impossible.
 */
int __printk_ratelimit(int ratelimit_jiffies, int ratelimit_burst)
{
	static DEFINE_SPINLOCK(ratelimit_lock);
	static unsigned long toks = 10 * 5 * HZ;
	static unsigned long last_msg;
	static int missed;
	unsigned long flags;
	unsigned long now = jiffies;

	spin_lock_irqsave(&ratelimit_lock, flags);
	toks += now - last_msg;
	last_msg = now;
	if (toks > (ratelimit_burst * ratelimit_jiffies))
		toks = ratelimit_burst * ratelimit_jiffies;
	if (toks >= ratelimit_jiffies) {
		int lost = missed;

		missed = 0;
		toks -= ratelimit_jiffies;
		spin_unlock_irqrestore(&ratelimit_lock, flags);
		if (lost)
			printk(KERN_WARNING "printk: %d messages suppressed.\n", lost);
		return 1;
	}
	missed++;
	spin_unlock_irqrestore(&ratelimit_lock, flags);
	return 0;
}
EXPORT_SYMBOL(__printk_ratelimit);

/* minimum time in jiffies between messages */
int printk_ratelimit_jiffies = 5 * HZ;

/* number of messages we send before ratelimiting */
int printk_ratelimit_burst = 10;

int printk_ratelimit(void)
{
	return __printk_ratelimit(printk_ratelimit_jiffies,
				printk_ratelimit_burst);
}
EXPORT_SYMBOL(printk_ratelimit);
```

全局的速度这个还是可以配置的
```text
[root@ming home]# cat /proc/sys/kernel/printk_ratelimit_burst
10
[root@ming home]# cat /proc/sys/kernel/printk_ratelimit
5
```




