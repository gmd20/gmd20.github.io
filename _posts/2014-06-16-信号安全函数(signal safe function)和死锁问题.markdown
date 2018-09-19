```text
    




  下载LOFTER我的照片书  |
看到微博上有人说到在signal handler里面使用localtime函数导致死锁的问题。
http://idning.github.io/twemproxy-deadlock-on-signal_handler.html

文章里面有一个总结:

Reentrant:
不使用全局变量.
不调用non-reentrant函数.
Thread-safe:
要求较低,
可以访问全局变量，不过需要加锁
每次调用它返回不同的结果也没关系
Async-Signal-Safe:
只有几个固定的函数是 signal-safe的
使用了锁的一定不是信号安全的（除非屏蔽了信号）
可重入函数一定是线程安全的

可重入函数一定是Async-Signal-Safe的.

很关键的一点就是内部使用了锁的函数都是不能在signal hanlder里面使用的，会导致死锁。
原因大概是，程序的某个线程获得了锁，然后然后还没释放，这时又被信号中断了，然后在signal handler函数里面又尝试去获取锁的话，就会重新获取锁，就死锁了。
localtime的内部实现就是因为里面需要用锁来做全局static共享变量的保护，所以最后就死锁了。看网上有好些人遇到这个问题了的。


这里有一个可以在信号里面使用posix函数列表。
https://www.securecoding.cert.org/confluence/display/seccode/SIG30-C.+Call+only+asynchronous-safe+functions+within+signal+handlers


这里有一个不是线程安全的posix函数的列表。
2.9.1 Thread-Safety
http://pubs.opengroup.org/onlinepubs/009695399/functions/xsh_chap02_09.html#tag_foot_1
localtime没出现在这个列表里面，但应该也不是线程安全的，共用了全角的static 变量了， 不过有一个线程安全的实现localtime_r
参考说明http://linux.die.net/man/3/localtime


signal的man page上面列出了所有的信号安全的函数，只有这些函数才能保证可以在signal handler里面是安全使用的。
Async-signal-safe functions
http://man7.org/linux/man-pages/man7/signal.7.html


看上去signal 还是不用的好啊，这么多限制，很容易出问题。以前也听同事说过。
什么线程cancel 机制，最好也是不用的好。


```
