```text
用来解决经典问题“persistently full buffer problem,” “bufferbloat”（http://queue.acm.org/detail.cfm?id=2071893）
AQM (active queue management) 方面的一个算法？

腾讯的文章 “浅谈过载保护”
http://djt.qq.com/article-156-1.html

Some queuing theory: throughput, latency and bandwidth
http://www.rabbitmq.com/blog/2012/05/11/some-queuing-theory-throughput-latency-and-bandwidth/

都对这种现象进行了解释， “浅谈过载保护”最后提到的一个解决办法
-----------------------------------------
　对于“每个系统要有能力发现哪些是有效的请求，哪些是雪球无效的请求”，这里推荐一种方案：在该系统每个机器上新增一个进程：interface进程。Interface进程能够快速的从socket缓冲区中取得请求，打上当前时间戳，压入channel。业务处理进程从channel中获取请求和该请求的时间戳，如果发现时间戳早于当前时间减去超时时间（即已经超时，处理也没有意义），就直接丢弃该请求，或者应答一个失败报文。
------------------------------------------
其实跟这个 CoDel (Controlled Delay Management) 算法有类似的思想吧。


acm queue的原文在这里。
Controlling Queue Delay
 by Kathleen Nichols, Van Jacobson | May 6, 2012
https://queue.acm.org/detail.cfm?id=2209336 
简单的就是测量所有packet从被加入队列的时间和被出去队列的时间只差，这个逗留时间如果大于target时间的情况在interval时间间隔内出现过，就通过丢包来控制队列的长度。
target 和interval是这个算法的两个参数，
 target就是正常的请开下，允许一个元素呆在队列里面的时间。
interval   最坏的情况下，处理一个消息或者包的时间。TCP网络对应最坏的可以接受的RTT。 如果包在队列里面时间超过这个interval 就要丢包，再处理也没有意义了。参考“浅谈过载保护” 等文章的解释。
为了保证平滑，丢包的时间速率计算比较有意思。

老实说算法文档英文看的不是很懂啊，不过给的个伪码的实现，一看就应该清楚了。
http://queue.acm.org/appendices/codel.html 



Linux内核开发者已经开始把这个算法用于  TCP的拥挤窗口控制了。 paper的作者最初应该也是用于linux路由器的tcp上面的。
http://lwn.net/Articles/496250/
http://lwn.net/Articles/496509/


RabbitMQ也把这个算法应用于消息队列的延时控制了
Some queuing theory: throughput, latency and bandwidth
http://www.rabbitmq.com/blog/2012/05/11/some-queuing-theory-throughput-latency-and-bandwidth/
```
