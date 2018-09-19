    

Pantheios 的网页说他的库比其他的 日志库要快很多，看这里的http://www.pantheios.org/related_material.html
的解释，大概是说，log startment都是使用栈上的内存吧，只是在log被启用的时候，才有最多一个的日志参数的复制。其实就是使用 一个  suto_bufer的辅助类，在栈上留一段空间，比如2048字节。 然后那些 log信息字串都是放到这个 栈上。 只有参数太多，格式化出来的字符串太长的时候才有 suto_bufer 去堆里面动态申请内存。 这样写日志的时候大多都是没有动态内存申请操作性能就比较高。
       
关于这个auto_buffer的解释，给的参考是： 写的也比较清楚。
 Efficient Variable Automatic Buffers
http://collaboration.cmc.ec.gc.ca/science/rpn/biblio/ddj/Website/articles/CUJ/2003/0312/0312wilson/cuj0312wilson.htm

不过看这个看下面这个.stlsoft的文档，举的例子就很清楚了。其实就是把 栈上类似的char [2048] 这样的局部变量给封装一下，便于使用而已。如果这个缓存不够用，再去用new 动态申请。
auto_buffer 
http://www.stlsoft.org/doc-1.9/

boost里面也有一个类似的实现。
http://www.boost.org/doc/libs/1_46_0/boost/signals2/detail/auto_buffer.hpp

简单的说，就是避免 写每一条日志时的动态分配内存和释放。 不过这有个问题，如果trace语句比较多，嵌套的话，可能占用栈比较多，在栈空间比较小的环境下，可能要注意。比如在linux内核8k的栈，这样子搞很容易就溢出了。那些格式化的开销看上去是不能避免了。Pantheios看起来和现在的接口整合起来不太方便。

网上有陈硕写的 muduo网络库里面用到的一个日志库
muduo's c++ high-perf logging
http://xanpeng.github.com/programming/2012/06/18/muduo-logging.html
他的logstream应该也是直接在栈上的内存，避免动态内存分配。这里有代码。
https://github.com/chenshuo/recipes/tree/master/logging

当前项目，有个trace的实现，用来辅助记录代码流程，性能很差，看了一下，那些string的日志，每次日志都需要动态内存分配，写完还要free。性能分析发现这个确实代价比较大啊。希望可以改写一下避免这种动态内存分配。 
