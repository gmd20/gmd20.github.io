程序crash之后，现在i配置默认启动 visutal studio 2008 visual studio 2010的调试器。并不直接生成dump文件或者error report。

这个应该是注册表里面的调试器设置相关的一个设置，之前也写文章了。



其实在visual studio里面选择 菜单 "Debug"  最后一项->  “Save Dump As” 

那里就可以生成dump文件，里没有  “mini dump ”  “mini dump with heap” 两种类型可以选择。保存就可以了。



Windows 2008的任务管理 里面可以 右键进程  -》 create dump  也可以为一个进程生成dump文件吧。
如果机器一开始没有配置生成dump文件，只有error report报告，可以用这个办法在同样的环境生成一个dump文件。
然后再分析dump文件，根据  error report里面 fault offset ，在出错的模块的基地址加上这个偏移就可以找到对应出错位置的汇编。
