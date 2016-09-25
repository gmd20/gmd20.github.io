IE11的保护模式（protect mode）导致程序FindFirstUrlCacheEntry不到IE缓存的问题
=========================================================================

以前写过一个小程序，提取IE缓存文件里面的mp3和视频，  mp3还自动解析文件里面ID3Tag的专辑名字等内容。

最近发现在windows10下面没法用了。看了一下，原因是 后面版本的IE 有protect mode。

这个东西是为了避免恶意软件的，也就是说IE进程已经 flash控件这种运行在  low模式，能访问的资源受限制了。

这样系统也区分开了，这些low模式的缓存存放在low目录下，跟其他正常的区分开了。

```
C:\Users\<user name>\AppData\Local\Microsoft\Windows\INetCache\Low\IE
```

WinInetAPI的FindFirstUrlCacheEntry这些接口也只有运行在low模式的才能遍历到这个缓存了。

所以正常应用要访问这些缓存，必须fork创建一个low模式的进程。

但low模式的程序又不能往普通的磁盘位置写文件，这样 low模式程序找到缓存的文件了，只能通过socket把这些文件名或者其他信息发送给 非low模式的代理程序，最后由代理程序来写文件。

 


参考文章：
========
 
 

A Primer on Temporary Internet Files
https://blogs.msdn.microsoft.com/ieinternals/2011/03/19/a-primer-on-temporary-internet-files/

Writing Files from Low-Integrity Processes
https://blogs.msdn.microsoft.com/ieinternals/2010/08/26/writing-files-from-low-integrity-processes/

Designing Applications to Run at a Low Integrity Level
https://msdn.microsoft.com/en-us/library/bb625960.aspx

Understanding and Working in Protected Mode Internet Explorer
https://msdn.microsoft.com/en-us/library/bb250462(v=vs.85).aspx

Protected Mode Broker Functions
https://msdn.microsoft.com/en-us/library/cc848890(v=vs.85).aspx

IEShowSaveFileDialog function
https://msdn.microsoft.com/en-us/library/ms537318(v=vs.85).aspx


Create low-integrity process in C# (CSCreateLowIntegrityProcess)
https://code.msdn.microsoft.com/windowsapps/CSCreateLowIntegrityProcess-d7cb5e4d/sourcecode?fileId=52580&pathId=6398360

Handle IE Protected Mode in managed code (c#)
http://blog.icnet.eu/handle-ie-protected-mode-in-managed-code-c/

A Developer's Survival Guide to IE Protected Mode
http://www.codeproject.com/Articles/18866/A-Developer-s-Survival-Guide-to-IE-Protected-Mode
