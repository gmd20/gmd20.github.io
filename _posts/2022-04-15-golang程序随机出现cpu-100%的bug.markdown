https://github.com/golang/go/issues/51654    
v1.16.7 ~ v1.18 都会出现这个bug。线上的机器遇到的好像就是这个问题，golang的应用一天中随机出现几次cpu 100%的问题，过几秒可能就恢复正常.
看描述好像说是在不同的goroutine里面修改同一个Timer场景会导致这个问题。

用perf采样，cpu利用率调用栈大概是这样的。
```text
  32.70%        [.] runtime.stealWork
  12.51%        [.] runtime.netpoll
   5.36%        [.] runtime.findrunnable
   4.51%        [k] sysret_check
   4.30%        [.] runtime.epollwait.abi0
   4.08%        [.] runtime.checkTimers
   4.01%        [.] __vdso_clock_gettime
   3.03%        [k] _raw_spin_lock_irqsave
   2.41%        [.] runtime.checkTimersNo
```

v1.19才会修复，有的等了。



v1.19发布了， v1.18.5也经过backport修复这个bug了吧 https://github.com/golang/go/issues/53847 
