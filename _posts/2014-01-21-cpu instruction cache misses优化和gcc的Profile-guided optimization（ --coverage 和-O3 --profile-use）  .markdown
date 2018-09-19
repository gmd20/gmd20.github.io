```text
    




  下载LOFTER我的照片书  |
40% better single-threaded performance in MariaDB
http://kristiannielsen.livejournal.com/17676.html

微博看到有人提到这篇文章，还是有点料的
首先有perf 观察instruction cache misses。
原文提到的这个技巧很有参考价值，以前没怎么注意这个 level 1 instruction cache ("icache").，看来这个东西对性能有很大的影响。一点点的比例对性能影响就很大。 vc 2013内置的profiler和 intel vtune好像也都有这个事件的。不过一直不怎么留意这个东西。明天测试一下看看这个 各个 指令cache miss  ， data 的cache miss的比例怎么样，看看能不能学到什么东西。

I am using the Linux perf program for this. During my invistigations, I found that the main bottleneck in the benchmark turns out to be instruction cache misses. Here is an example measurement:
Counter	Value
INST_RETIRED.ANY	170431
ICACHE.MISSES	17194
So 10% of executed instructions missed the level 1 instruction cache ("icache"). That is bad. The Intel Optimization Manual says this about the ratio ICACHE.MISSES/INST_RETIRED.ANY:
Anything over 1% of instructions retired can be a significant issue.
So we are 10 times worse than "a significant issue".
Instruction cache misses cause a bottleneck in the frontend of the CPU - where x86 instructions are fetch, decoded, and supplied to the micro-op queue to be scheduled for the out-of-order dataflow backend. To get an idea about how badly bottlenecked we actually are in the frontend in this benchmark, we can use another measure from the Intel manual:
IDQ_UOPS_NOT_DELIVERED.CORE / (4*CPU_CLK_UNHALTED.THREAD) = 81.5%
This ratio estimates the percentage of time in which the front-end is not able to deliver instructions sufficiently fast for the backend to work at full speed. In this case, for more than 80% of the time, the CPU would have been able to do more work in the same time if only instructions could be fetch and decoded faster.

然后作者尝试了一下
Profile-guided optimization
https://en.wikipedia.org/wiki/Profile-guided_optimization

就可以降低 L1 instruction cache miss 2% 读操作性能就提高了40%
其实就是利用gcc 的 -O3优化里面一个选项吧
先设置CFLAGS/CXXFLAGS. 加上 --coverage ， 之后编译执行一遍，应该是采集结果到profile文件。
然后再次用-O3 --profile-use 选项编译程序，应该就利用到前面一次执行的profile结果，重新调整代码组织结构，生成cache更友好的程序吧。 

不过这个-O3 好像不推荐使用啊，前面有论文说 debian里面的c/c++代码用-O3 编译大部分都有bug。

感觉这个文章作者很喜欢玩这些技巧，比如前面这篇。
MySQL/MariaDB single-threaded performance regressions, and a lesson in thread synchronisation primit
http://kristiannielsen.livejournal.com/17598.html

不过是不是真正有用就有待观察了，了解这些技巧也不错！
```
