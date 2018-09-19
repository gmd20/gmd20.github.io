    




  下载LOFTER我的照片书  |
一直对这个问题似懂非懂的样子，再找了一些文章看看。



False sharing occurs when multiple concurrent tasks that are running on separate processors write to variables that are located on the same cache line. When one task writes to one of the variables, the cache line for both variables is invalidated. Each processor must reload the cache line every time that the cache line is invalidated. Therefore, false sharing can cause decreased performance in your application.



1.避免false sharing的办法

  1）减少写的次数，先在线程局部变量里面缓存结果，最后才一次性更新写入。

  2）再发生false sharing的两个数据之间插入间隔，人为的把两个数据分离到两个不同的cache line上面。

     鉴于cache line 大小一般为64，所以一般在中间插入64个字节

     比如让两个int 对齐

     const int CACHE_LINE_SIZE = 64; 

     char pad1[CACHE_LINE_SIZE - sizeof(int)];; 

     或者使用编译器的对齐原语

     __declspec (align(64)) int element;

  3）尽量使用线程私有数据。 不同的线程、按照读和写分开数据避免共享很重要！

  4） 利用编译器优化，减少内存读写操作memory loads and stores。



2. 一个处理器写，另外一个在读同一个cache line也会受影响。可能没有两个cpu同时写一个cacheline影响大？

也是有影响的。参考这篇文章。

Eliminate False Sharing By Herb Sutter, May 14, 2009

http://www.drdobbs.com/parallel/eliminate-false-sharing/217500206?pgno=1





3. false sharing的地方可以被硬件性能检测事件查看出来（普通的指令周期的应该也可以看的出来）。 内存操作和memory l1 l2 cache miss 比较高？

false shang问题的本质就是不同cpu上同一个cache line的交互更新，导致cache同步协议MESI的通讯成本的增加。

参考这篇文章 Avoiding and Identifying False Sharing Among Threads

http://software.intel.com/en-us/articles/avoiding-and-identifying-false-sharing-among-threads



4. lock前缀的原子操作。 这种atomic 数据操作同一个数据，肯定也是在同一个cache line 上面，如果多个cpu上的线程同时更新，肯定也有false sharing一样的共享问题。

   所以本质上spinlock之类使用了lock前缀的指令，其实也是 一种sharing problem。但这种共享是“明显”的，所以大家一般不说“false” sharing，难道说true sharing？

   可能false sharing 多是指不那么明显发现的问题吧，比如不同的变量处在同一个cache line。

   http://stackoverflow.com/questions/10143676/false-sharing-and-atomic-variables  

   http://www.1024cores.net/home/lock-free-algorithms/first-things-first

   根据上面两个文章知道，atomic 的共享访问也是会有false sharing一样的问题来的。atomic 访问如果不是多cpu同时更新，和普通的写更新是一样快的, 但如果多个进程在不同的cpu上面更新就退化的很快了。大多都是耗在这种多cpu的更新同步上面，所以atomic 的lock的更新的曲线和false sharing更新的性能退化曲线几乎是一样的。

   根据intel的文档，lock前缀的指令，很多时候都可以避免bus lock，而是使用这种cpu的cache同步协议来实现的。看来bus lock应该代价更大。那么可以确定是原子操作lock这种同样具有false sharing一样的duocpu共享写一个cache line的问题。



     

5. false sharing， 只有线程把大量的时间都花在这个引起问题的写操作上的时候，写操作很多的时候，才会引起足够大的值得关注的问题。大多情况下比如更新atomic变量，这些应该不会有问题吧？

   看来对false sharing问题也不能太敏感啊。我的一个代码里面，使用了atomic变量，也会两个线程写的同步，但看上去通过避免了很多内存分配和释放操作，看上去这个获得的性能比起引入false sharing问题，总体还是好了很多。





6. false sharing看到一个文档的测试，导致性能慢十倍。

