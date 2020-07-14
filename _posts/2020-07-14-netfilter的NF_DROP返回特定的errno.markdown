```text
/* Returns 1 if okfn() needs to be executed by the caller,
 * -EPERM for NF_DROP, 0 otherwise.  Caller must hold rcu_read_lock. */
int nf_hook_slow(struct sk_buff *skb, struct nf_hook_state *state,
		 const struct nf_hook_entries *e, unsigned int s)
{
	unsigned int verdict;
	int ret;

	for (; s < e->num_hook_entries; s++) {
		verdict = nf_hook_entry_hookfn(&e->hooks[s], skb, state);
		switch (verdict & NF_VERDICT_MASK) {
		case NF_ACCEPT:
			break;
		case NF_DROP:
			kfree_skb(skb);
			ret = NF_DROP_GETERR(verdict);
			if (ret == 0)
				ret = -EPERM;
			return ret;
		case NF_QUEUE:
			ret = nf_queue(skb, state, s, verdict);
			if (ret == 1)
				continue;
			return ret;
		default:
			/* Implicit handling for NF_STOLEN, as well as any other
			 * non conventional verdicts.
			 */
			return 0;
		}
	}

	return 1;
}
EXPORT_SYMBOL(nf_hook_slow);
```
通常，如果hooks里面如果返回NF_DROP之后，看上面的代码，它是返回EPERM给应用的，sendto那些函数看到的就是errno就是EPERM了。
但hooks其实也可以通过这个宏 #define NF_DROP_ERR(x) (((-x) << 16) | NF_DROP) 返回特定的错误码 ，看了上面的代码才发现这个用法。
另外hooks返回NF_STOLEN，又直接kfree_skb 丢掉包，应用层应该是没法知道了，因为如上面代码所示它返回0给上层。


