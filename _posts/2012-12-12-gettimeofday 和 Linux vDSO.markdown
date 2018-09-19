    




  下载LOFTER我的照片书  |
      看到微博里面说道gettimeofday的优化问题，如果程序需要调用?? ????gettimeofday 太频繁的话，都是设置定时器，比如一毫秒的，获取到时间保存下来作为cache提供，这样可以减少?? gettimeofday的数量。这个之前就了解了。

      还提到 Linux  vDSO 的，其实我之前也一直以为是个  系统调用的，但其实由于 Linux  vDSO （Virtual Dynamically-lined Shared Object)，其实这个内核函数已经映射到用户空间了，虚拟了一个so共享模块，调用起来应该和直接调用用户空间 的c函数一样的了，没有系统调用的开销了。  在Linux系统上面 自己查看程序的so文件映射应该卡伊看到这个vdso的的。所以gettimeofday的代价应该比较小的了，之前有同事说他测试过确实还是很快的。

          比较有意思的还有，这个vdso虚拟模块地址，为了安全起见，那个映射地址应该是随机的，避免hook



内核中vdso的简单文档和导出的函数，可以看到主要是几个时间函数。

http://lxr.linux.no/linux+v3.7/Documentation/ABI/stable/vdso

http://lxr.linux.no/linux+v3.7/arch/x86/vdso/vdso.lds.S



 其实类似的优化机制还有vsyscall，都是把内核空间的东西直接映射给用户空间使用，避免系统调用的开销。

On vsyscalls and the vDSO  

http://lwn.net/Articles/446528/



涉及vdso优化?gettimeofday的原理和如何自己添加vdso函数，很不错的文章

Creating a vDSO: the Colonel's Other Chicken

http://www.linuxjournal.com/content/creating-vdso-colonels-other-chicken



csdn博客里面一篇文章

http://blog.csdn.net/wlp600/article/details/6886162
