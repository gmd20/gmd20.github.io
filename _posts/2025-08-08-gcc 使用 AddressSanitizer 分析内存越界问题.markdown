有程序 SIGBUS 信息退出了，应该是内存越界问题，这种都是事后不在第一现场了，不好定位到源码越界地方。
还好，现在有gcc这些都集成了 “AddressSanitizer” ，比较好用。

dnf install libasan  安装libasan 库，    
加上编译选项  " -fsanitize=address " 编译程序，运行出错就会打印内存越界时的函数调用链了。   
