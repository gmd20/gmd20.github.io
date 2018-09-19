     在这里http://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/index.html 看到 ftrace的实现原理，觉得很有意思的用法。

      linux 内核里面配置了开启ftrace之后，利用 gcc  的 -pg 选项，让每个内核函数的开头自动插入调用 mcount函数的代码。gcc 的 -pg选项是为了性能测试用的，但不知道ftrace还能利用到这点。ftrace实现了自己的mcount函数，这样就可以在每个内核函数调用到的时候i进行性能统计。

       ftrace 利用 ringbuffer来管理各个tracepoint，为了不影响性能也支持dynamic trace，就是运行时的trace开关，就是把mcount定义为空的，在系统运行起来，用户配置配置了指定的trace点后才启动trace开关。  不然所有的内核函数入口都先跑这个ftrace应该比较影响性能。

       相比较 kprobe的在代码里面直接插入 int 3的实现，这个ftrace也有他自己的特点啦。Linux内核文档有几个ftrace相关的，自己去看一下吧。

http://lxr.linux.no/linux+v3.5/Documentation/trace/ftrace.txt
