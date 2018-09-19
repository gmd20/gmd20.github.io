```text
C++ memory profiler  检查c++ 程序内存都消耗在什么地方

Analyzing the Performance Impact of Memory Utilization and Garbage Collection
http://javabook.compuware.com/content/memory/analyzing-java-memory.aspx
好像这些带GC的语言分析内存问题比较方便。

C++我很想有一个这样的工具可以使用，比如可以在运行时attach到某个进程，然后可以看各个不同的object size都分配了多少个，
占用的比例有多大，什么地方调用的，甚至可以查看某个size的内存块里面的内容是什么，然后最好还可以可视化，比较直观的显示结果。

好像有些收费的商业工具符合我的要求
http://www.softwareverify.com/cpp-memory-object.php
http://www.puredevsoftware.com/MemPro.htm
http://mtuner.net/


不过收费的实在没什么兴趣啊，还好有很多免费的可以选择。

之前考虑研究一下systemtap这种结合 tmalloc的代码，实时打印各个size的object个数和内存，应该可以做的到的吧。


visual studio 里面提供
CRT Debug Heap Details
http://msdn.microsoft.com/zh-cn/library/974tc9t1.aspx
看上去很不错，使用 _CrtMemDumpStatistics  _CrtMemDumpAllObjectsSince _CrtDumpMemoryLeaks 这类函数硬
但只能在debug模式使用这个就有点不爽啊，不是每个项目都方便在debug下面运行的。


tcmalloc 的 Heap Profiler
https://code.google.com/p/gperftools/wiki/GooglePerformanceTools
可以打印所有内存分配的在各个函数里面占用比例，可以输出图形


Valgrind 的 Massif  有人相比较 google pertf tool的Heap Profiler这个性能比较差。
http://valgrind.org/docs/manual/ms-manual.html


Dr. Memory
http://www.drmemory.org/
之前用过一个Windows平台的内存检查工具，也有类似的功能？
-top_stats  -no_check_leaks  -no_count_leaks -light
```
