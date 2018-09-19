关于Memory Barrier，理解并行编程必须掌握的一个知识点吧，但用户编程好像接触的不多，至少我没发现那个语言的参考书有说这个的。

 （2012-2-24 补充， visual c ++ 里面比较新的版本有支持了，

硬件的Memory Barrier 有宏 void MemoryBarrier(void);
软件的 编译器提示的Memory Barrier  的有宏 _ReadBarrier, _WriteBarrier, and _ReadWriteBarrier 
http://msdn.microsoft.com/en-us/library/f20w0x5e(v=vs.100).aspx
http://msdn.microsoft.com/en-us/library/windows/desktop/ms684208(v=vs.85).aspx

）

( 2012-03-12 补充 C++ 11 里面的 std::atomic 的几个std::memory_order属性应该也是和这个有关的http://en.cppreference.com/w/cpp/atomic/memory_order )

         一般的linux内核编程图书应该都有介绍，但一般说的都不是很仔细，发现讲的最多的解释的来自下面这本书。基本 memory order和cache相关的知识都讲的很细，然后举了很多例子。不过我都没耐心看完，现在也还一知半解的。

 

Paul E. McKenney  "perfbook.2011.08.28a.pdf" (Is Parallel Programming Hard, And, If So, What Can You Do About It?)

 

2.1.4 Memory Barriers

12.2 Memory Barriers

Appendix C Why Memory Barriers?

Paul E. McKenney 另写一个单独的文档，详细说了cpu 的cache 和 Memory Barriers，跟上面内容有点重复，可以看一下。

Memory Barriers: a Hardware View for Software Hackers

 http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.06.07c.pdf

--------------------

Intel的手册这个也值的一读。

Intel?64 and IA-32 Architectures

Software Developer抯 Manual

Volume 3A:

System Programming Guide, Part 1

 8.2 MEMORY ORDERING

-------------------------------------

网友翻译的 linux内核文档 Documentation/memory-barriers.txt ，非常的不错，也是Paul E. McKenney  等额人写的，可能直接去看这个比较合适吧。这个上面文档内容有点重复，不过这个好像更清晰一点。建议先看这个吧。 2012-03-07

http://blog.chinaunix.net/space.php?uid=9918720&do=blog&id=1640912
