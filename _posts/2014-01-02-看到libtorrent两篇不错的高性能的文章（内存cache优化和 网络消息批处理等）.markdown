```text
    




  下载LOFTER我的照片书  |
1. 

memory cache optimizations

http://blog.libtorrent.org/2013/12/memory-cache-optimizations/

又是使用mprotect，和用户态的SIGSEGV  signal handler来处理缺页异常（可以参考下面的access_profiler项目源码学习一下）。然后记录所有的类的成员的访问内存的偏移，这样就可以把类里面所有的成员访问的次数进行可视化。两个工具结合起来就可以找那些成员是经常访问的，可以把频繁访问的成员调整到一块，放到最前面的cache line里面去，达到优化的目的。 

文章里面提到两个辅助项目，

https://github.com/arvidn/struct_layout

https://github.com/arvidn/access_profiler

类似一个命令

pahole，可以把gcc生成的调试符号的文件里面  类型打印出来。 按照cache size对齐那些。没有用过，可以试试

http://linux.die.net/man/1/pahole





2. 

principles of high performance programs

http://blog.libtorrent.org/2012/12/principles-of-high-performance-programs/



context switching

batching work

今天才和同事讨论了一下批量发送网络消息可以减少syscall的次数的优化。这个文章也是说这个的，说是batching work，可以减少syscall导致的context switching，以及怎么触发信号唤醒线程，才能避免更多的开销。

比如等所有的消息都处理完了，全部加到消息队列里面去了，才去唤醒另外一个线程，不要每次生成一个消息加入队列都去唤醒线程一次，减少这种重复wake-up之类的开销。之前我的代码好像也做过类似的实现。

下面的文章里面提到的技巧：可以去看一下原文，写的很不错，还是使用boost::asio来举的例子。

Always complete all work queued up for a thread before going back to sleep.

When awake, have your thread complete all its work before committing the work that it produced.

Allocate memory buffers up front to avoid extra copying and maintain predictable memory usage.

When determining your batch size, think about what a natural division is without using magic numbers. It often involves the number of jobs (or bytes) one accrues during the time it takes for your thread to wake up after having been signaled, or during one scheduler time slice.

The principle of organically adapt your job-batching sizes to the granularity of the scheduler and the time it takes for your thread to wake up.

A mechanism to adapt receive buffer sizes for sockets, while at the same time avoiding to copy memory out of the kernel.

Suggestions on how an operating system (and any API) can be shaped to better scale its performance

Only signal a worker thread to wake up when the number of jobs on its queue go from 0 to > 0. Any other signal is redundant and a waste of time.

Cork sockets in order to merge as many writes as possible

A pattern to support committing-work-when-queue-is-drained to a high-level message queue


```
