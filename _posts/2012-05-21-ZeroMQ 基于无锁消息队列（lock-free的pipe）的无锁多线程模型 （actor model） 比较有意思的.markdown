
 原文地址： 
ZeroMq的架构文档
http://www.aosabook.org/en/zeromq.html
强烈建议读一下这个吧

在线源码：
https://github.com/zeromq/libzmq/tree/master/src


1.
i_engine.hpp  -> stream_engine.hpp
有多种继承自i_engine的类
调用socket api的代码就是在这里面。

2.
socket_base.hpp  ->  xsub_socket_t 
session_base.hpp  -> xsub_session_t   (xsub.hpp   pair.hpp  push.hpp  pull.hpp ...)
有多种继承自 socket_base 和session_base的 类。
不同socket模式，主要是根据 pipe的对应关系区分，比如一对多，多对一的pipe对的维护和管理等。
发送和接受时的群发的实现的就在这些相关的类里面实现。必须管理多个pipe对等。
什么负载均衡啊也是在这一层里面做的。具体可以看上面实现类里面的对 lb.hpp dist.hpp  trie.hpp等类的实现，看看是怎么管理多个pipe对的。

3.
yqueue.hpp ->     ypipe.hpp ->     pipe.hpp 
不同的对象通过 pipe 来相连的，pipe的两端注册有pipe event异步通知事件，read 或者 write东西到pipe的时候，自动触发 异步读写事件。 pipe是一个lock-free的 队列，类似 单生产者单消费者的无锁队列，但支持批量消息的入队和出队操作。所有跨线程间的交互都是通过往无锁队列里面写入消息来进行的，得益于这个无锁pipe的使用，整个系统完全的无锁设计， 符合“actor model.” 模型的原理。  关键就是msg_t 结构在pipe里面的传输和传输过程所触发的pipe event。   不同对象，比如socket_t和 后端的 i_engine就通过这个pipe关联起来。


4.
其他的比如message的批量发送， message的内存管理等也比较有意思。根据他们的测试，下面这三个是比较重要的影响性能的因素
Number of memory allocations    message的引用计数，和copy还是new的选择。
Number of system calls           批量发送message可以减少系统调用的次数。
Concurrency model            上面所说的无锁模型actor model.“。
