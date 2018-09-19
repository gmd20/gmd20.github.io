```text
很早之前就看这个系列的文档了 “What every programmer should know about memory” 应该是redhat的一个工程师写的。不过英文的版的，篇幅很长，不怎么认真看过。



今天看到开源中国上面有人有翻译成中文版了

每个程序员都应该了解的 CPU 高速缓存

http://www.oschina.net/translate/what-every-programmer-should-know-about-cpu-cache-part2



翻译的很不错！！！



原文是在LWN上的，不过之前也有见过整理成一个pdf。今天有时间又去看了一下下面几个小节。

Memory part 2: CPU caches

http://lwn.net/Articles/252125/

Memory part 5: What programmers can do

http://lwn.net/Articles/255364/

Memory part 6: More things programmers can do

http://lwn.net/Articles/256433/

Memory part 7: Memory performance tools

http://lwn.net/Articles/257209/



好的文档总是要多次强化复习啊，每次都有收获，读的越多越感觉是写的好。

What programmers can do  和 More things programmers can do 里面讲的是程序员怎么才能让程序更充分利用的cpu cache。

讲到的多线程同时写一个cache line导致的false sharing 问题，（只有多个cpu之间同时写一个cache，才需要在cpu之间发RFO消息同步MESI状态，导致性能变差。只是读的话，直接读cache里面的旧的内容，不需要同步RFO请求，所以没有这个问题。所以多线程只读的常量那些可以放到一块的，编译器也是这么干的）。

        文档给的一些建议怎么组织 多线程同时read 和write的数据，来避免这个false sharing问题：

In general the best advice which can be given is



Separate at least read-only (after initialization) and read-write variables. Maybe extend this separation to read-mostly variables as a third category.

Group read-write variables which are used together into a structure. Using a structure is the only way to ensure the memory locations for all of those variables are close together in a way which is translated consistently by all gcc versions..

Move read-write variables which are often written to by different threads onto their own cache line. This might mean adding padding at the end to fill a remainder of the cache line. If combined with step 2, this is often not really wasteful. Extending the example above, we might end up with code as follows (assuming bar and xyzzy are meant to be used together):int foo = 1; int baz = 3; struct { struct al1 { int bar; int xyzzy; }; char pad[CLSIZE - sizeof(struct al1)]; } rwstruct __attribute__((aligned(CLSIZE))) = { { .bar = 2, .xyzzy = 4 } };

Some code changes are needed (references to bar have to be replaced with rwstruct.bar, likewise for xyzzy) but that is all. The compiler and linker do all the rest. {This code has to be compiled with -fms-extensions} on the command line.}

If a variable is used by multiple threads, but every use is independent, move the variable into TLS.



 

关于NUMA架构的介绍也是我看到最详细的资料。

Memory part 5: What programmers can do 里面介绍的  cache bypass 更新策略，cache prefetch等知识也好很有用。还讲到的很多组织数据和代码，获取很好的指令缓存利用和 分支预测，怎么优化page fault等等

总的来说，程序使用的内存（working set） 是最小最好了，能全部装在L1 cache 肯定比要用到 L2 cache L3 cache等要好的多。文档比较了不同线程，不同working set的性能评测。



确实是好东西来的，每个程序员都应该看一下这个文档！！！！！！
```
