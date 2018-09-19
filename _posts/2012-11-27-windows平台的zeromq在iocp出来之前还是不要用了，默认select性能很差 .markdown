默认在Windows平台用的select ，性能很差！在事件循环里面调用select之前初始化fd_set的memcpy消耗额外的16%的cpu资源。



之前用了zeromq做了个传输log的后端，发现 libzmq.dll 占用的cpu比较多，用性能工具分析发现，原来是select的问题。

windows平台的zeromq在iocp出来之前还是不要用了，默认select性能很差 - widebright - widebright的个人空间
 


可以看到这个地方的3个memcpy的占用非常的多。之前一直怎么了解select的性能为什么比较差。看到这个终于清楚了。select函数

http://msdn.microsoft.com/ZH-CN/library/windows/desktop/ms740141(v=vs.85).aspx

要求 这几个输入参数每次都需要初始化，所以这几个memcpy不能避免。



The problem with select()

http://static.usenix.org/event/usenix99/full_papers/banga/banga_html/node3.html 

这里对这个问题有很详细的解析。



Poll 函数

http://static.usenix.org/event/usenix99/full_papers/banga/banga_html/node4.html

避免了这个每次都要初始化的问题，所以性能应该是要的。epoll那些也都没有这个初始化的问题。可以看看 zeromq里面的集中polerl的实现的loop函数

https://github.com/zeromq/libzmq/blob/master/src/select.cpp

https://github.com/zeromq/libzmq/blob/master/src/poll.cpp

https://github.com/zeromq/libzmq/blob/master/src/epoll.cpp



都没有select的这个问题，windows平台没有pol的实现，iocp好像还在开发中？ 不知道在哪个zeromq的l版本才能出来。

select的这种性能是不能接受的吧，额外的消耗多了16.5%的cpu啊！



用个asio来重写一下看看吧，其实我也不怎么喜欢 asio，好像每次操作的 handler对象的复制代价也有点大。没办法，自己直接用api更麻烦，本来就看中zeromq的api简单，想不到在windows平台做的不好啊
