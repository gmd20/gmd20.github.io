```text
参考文档
http://google-perftools.googlecode.com/svn/trunk/doc/heapprofile.html
http://goog-perftools.sourceforge.net/doc/heap_profiler.html

在debian 7的虚拟机上面测试，



libtcmalloc-minimal4 包是不包含heap profiler之类的工具在里面的吧
普通的内存分配可以用 libtcmalloc-minimal4，
但如果使用profiler就要用libgoogle-perftools4 这个包？
google-pprof 这个分析profiling结果的在   google-perftools 包。
------------------------
root@I# dpkg-query  -L libtcmalloc-minimal4
/usr/lib/libtcmalloc_minimal.so.4.1.0
/usr/lib/libtcmalloc_minimal.so.4
root@# dpkg-query -L google-perftools
/usr/bin/google-pprof    // profiler结果查看工具
root@# dpkg-query -L libgoogle-perftools4
/usr/lib/libtcmalloc_debug.so.4.1.0
/usr/lib/libtcmalloc_and_profiler.so.4.1.0
/usr/lib/libprofiler.so.0.3.0
/usr/lib/libtcmalloc.so.4.1.0
/usr/lib/libtcmalloc_debug.so.4
/usr/lib/libprofiler.so.0
/usr/lib/libtcmalloc.so.4
/usr/lib/libtcmalloc_and_profiler.so.4
apt-get install libgoogle-perftools4-dbg

----------
export LD_PRELOAD="/usr/lib/libtcmalloc_minimal.so.4"   使用minimal包中的 tcmalloc
export LD_PRELOAD="/usr/lib/libtcmalloc.so.4"   使用带完整profiler功能的so
export HEAPPROFILE=/tmp/tcmalloc_heap_profiler           启用heap profiler ， 生成的统计数据保存在/tmp目录下
启动你的程序
unset LD_PRELOAD   取消环境变量设置
unset HEAPPROFILE
cat  /proc/<pid>/maps     检查 so文件是不是正确加载了
cat  proc/<pid>/environ   查看环境变量设置
cat /proc/12411/status    查看内存

top -H 进去按M观察内存

注意如果是在终端里面export了这个变量的话，后面终端输入的命令默认都做profile了，
所以看到有的命令heap profiler失败，输出这些
root@# top -H
Starting tracking the heap

	signal 27 (PROF) was caught by top, please

可以在运行完要测试的程序之后，就unset <变量>
或者新开一个终端吧

直接这样运行，设置的环境变量只在这句其作用
LD_PRELOAD="/usr/lib/libtcmalloc.so.4"  HEAPPROFILE=/tmp/tcmalloc_heap_profiler   /usr/bin/freeDiameterd -c /opt/diameter/relay/freeDiameter.conf

google-pprof  --text /usr/bin/freeDiameterd-1.2.0 /tmp/tcmalloc_heap_profilera_15264.0039.heap
但发现显示的结果为空的，应该是 默认的包里面的libgoogle-perftools4 的heap profiler 用不了，stacktrace不正常。
查看/tmp/tcmalloc_heap_profilera_15264.0039.heap 这种导出来的内容，可以看到
 “ MAPPED_LIBRARIES: ”
 的模块映射之前的stacktrace内容为空。正常有很多行下面这样调用栈指针的记录的。
   “ 1:   229396 [     1:   229396] @ 0xb339a506 0xb339ac6a 0xb33b647d 0xb3391519 0xb3d56b96”

  google-pprof  这个perl脚本里面才能根据这些数据和调试符号得到 函数的统计。

 根据 gperftools-2.1 自带的文档  README和 INSTALL的说明，
 在64位系统要打开gcc 选项-fno-omit-frame-pointer 或者使用libunwind库才行。
 不知道默认的debian包在虚拟机上面为什么不行。

 自己在根据  gperftools-2.1 源码，编译一个，
 运行
./stacktrace_unittest
 ./heap-profiler_unittest
等单元测试都没有问题。
使用了自己编译的 libtcmalloc.so.4 之后， heap profiler
的stacktrace也正常了。看来是debian自带包和虚拟机的兼容问题，有可能一些gcc选项之类的不一样导致的？

之后再用下面这些命令查看结果就正常了。
google-pprof --gv /usr/bin/freeDiameterd-1.2.0 /tmp/tcmalloc_heap_profilera_15264.0013.heap 
google-pprof --text  /usr/bin/freeDiameterd-1.2.0 /tmp/tcmalloc_heap_profilera_15264.0013.heap 


bright@debian01:~/gperftools-2.1$ google-pprof --text --alloc_objects  /usr/bin/freeDiameterd-1.2.0 /tmp/tcmalloc_heap_profilera_15264.0099.heap | head -n 30
Using local file /usr/bin/freeDiameterd-1.2.0.
Using local file /tmp/tcmalloc_heap_profilera_15264.0099.heap.
Total: 308993762 objects
108017386  35.0%  35.0% 108017386  35.0% fd_msg_avp_new (inline)
70664963  22.9%  57.8% 70664963  22.9% os0dup_int
14134885   4.6%  62.4% 14134885   4.6% 0x00000000b360420b
14134885   4.6%  67.0% 14134885   4.6% 0x00000000b360424b
14123603   4.6%  71.5% 14123603   4.6% fd_fifo_post_internal
11312448   3.7%  75.2% 11312448   3.7% 0x00000000b35d0548
11312448   3.7%  78.9% 11312448   3.7% 0x00000000b35d0583
10894034   3.5%  82.4% 10894034   3.5% 0x00000000b35255d0
 5654146   1.8%  84.2%  5654146   1.8% std::basic_string::_Rep::_S_create
 5653958   1.8%  86.1%  5653958   1.8% 0x00000000b353d492
 2832646   0.9%  87.0%  2832646   0.9% fd_msg_new

--gv选项生成的图示，还有--callgrind  等格式的输出的，参考google-pprof （源码里面 pprof脚本） 的参数说明
```
