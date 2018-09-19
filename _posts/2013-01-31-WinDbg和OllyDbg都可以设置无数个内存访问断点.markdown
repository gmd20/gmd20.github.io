微博看到有人说c++ 内存被破坏，导致指针数组里面的某个指针值不错。感觉可以通过设置内存访问断点来查找问题。

不过说这个指针比较多，不能用这种办法。

试试vc2008里面的 数据断点，确实有个数限制，我的电脑只能设置4个，每个4个字节长度的断点。 再多就提示超过硬件限制了。

应该是x86里面4个调试寄存器，不能再多了。  （这个寄存器x86 debug register  有DR0...DR7  8个，但好像这有4个能用 http://en.wikipedia.org/wiki/X86_debug_register）



网上说是OllyDbg 这种调试器通过设置内存访问属性，可以下无数个内存访问断点。

不知道是不是通过VirtualProtectEx 函数来修改页的属性？ 

有时间在OllyDbg 里面试试



1月31号，补充一下

根据下面这篇文章，WinDbg也是通过改变页面属性来提供无数个内存读写访问的断点。

Memory Access Breakpoint for Large Ranges using VirtualProtect

http://blogs.microsoft.co.il/blogs/sasha/archive/2011/03/23/memory-access-breakpoint-for-large-ranges-using-virtualprotect.aspx

Visual Studio has had hardware-assisted memory access breakpoints for quite some time now—they are called data breakpoints and, unfortunately, can be used only for unmanaged code. WinDbg has a similar feature with the ba command, which is more flexible because it can be configured to fire when a memory location is read, not just written. (Of course, WinDbg does not have the managed/unmanaged limitation either, but you still have to exercise caution because managed objects may move in memory every time a garbage collection occurs, which would require you to reconfigure the hardware breakpoints.)

This type of breakpoint is implemented using hardware support from the CPU. On Intel platforms, the DR0-DR7 debug registers are used to enable hardware breakpoints on reading/writing memory locations. For example, to set a hardware breakpoint on reads and writes to four bytes at the address 0x01000100 you set DR0 to this address, and set bits 0, 16, 17, 18, 19 of DR7 to 1.

This is a great feature, and its only limitation is that the area watched by the breakpoint is at most 4 bytes (on 32-bit processors) or 8 bytes (on 64-bit processors)*. In other words, if you have a large array that is being corrupted in a random location once in a while, you can’t figure out the source of the corruption using hardware breakpoints.

What you could do, however, is the following trick**. Suppose for a moment that you large array resides precisely on page boundaries, e.g. an array of 4,096 integers that starts on a page boundary and ends after four pages (x86 pages are 4KB). Then what you could do is:

Temporarily change the virtual memory protection of that page to PAGE_GUARD.

Subsequent accesses (reads or writes) to that page will raise a special exception, a guard page violation.

Your debugger could catch this exception, verify that it occurred on access to that special address, and notify you that a breakpoint occurred.

When you want to continue execution, the debugger could allow the memory access to go through, set the PAGE_GUARD flag again, and proceed.

Although this is fairly tedious (or impossible) to accomplish in Visual Studio, it sounds like a great task for a WinDbg extension which I might write one day. In the meantime, we can use the SDbgExt extension which exports the !vprotect command, wrapping the VirtualProtectExWin32 API.

Here’s a sample debugging scenario with that idea in mind. The program being debugged is the following:



这里提到OllyDbg调试器就是用VirtualProtectEx 函数链实现的。

Extending windbg with Page Fault Breakpoints

http://www.codeproject.com/Articles/186230/Extending-windbg-with-Page-Fault-Breakpoints



这篇文章提到，OllyDbg的内存断点的实现原理

对抗OD内存断点

http://hi.baidu.com/haog8/item/0c05cd2ede4443d40f37f96d



1，OD内存断点原理
    A,内存访问断点，OD将目标内存所在的页面（范围圆整为1000h的倍数）设置为PAGE_NOACCESS，当被调试程序对这个内存进行任何“读、写或运行”操作时，都会触发异常。
    B,内存写入断点，OD将目标内存所在的页面（范围圆整为1000h的倍数）设置为PAGE_EXECUTE_READ，当被调试程序对这个内存进行“写”操作时触发异常。
    （原来都不是传说中的PAGE_GUARD?太惊讶了。

2，反OD内存断点原理
    要知道OD可以通过VirtualProtectEx改变内存页面属性，我们当然也可以修改，这样我们就可以发现内存断点或使其失效。



2月4号

类似的功能，在linux上面可以通过mprotect 系统调用和 libsigsegv 来实现。mprotect修改页面的属性，libsigsegv可以在用户空间处理page fault在满足条件的内存块的时候才中断下来。



微博有人发了文章 ， 说到怎么使用上面的这两个东西来查找c++的一个多线程内存越界问题，可以看一下。

定位多线程内存越界问题实践总结

 http://wenku.baidu.com/view/5b1a6616b7360b4c2e3f64fd.html  

