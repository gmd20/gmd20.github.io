使用bison的时候，发现很多有意思，使用gcc编译 c++的源文件的时候，提示的错误还是i显示的位置在  *.y 文件里面的。

觉得很有意思的一个功能。查看了一下代码，bison是在生成的代码里面插入了多  #line 来实现的。



gcc的文档 “6 Line Control ”

http://gcc.gnu.org/onlinedocs/cpp/Line-Control.html



这里有详细的说明。

#line linenum

#line linenum filename

#line anything else 

除了前面的两种，还可以使用任何可以展开的成前面两种默认的宏。



通过#line 修改后，后面的错误就报的错误在 #line指定的文件名里面的 。直到新的#line 指定新的文件名为止。



所以bison 生成的代码结构一般是这样子的：

------------------------

#line  123  “test.y”

插入语来自 test。y  的源码

#line  456  "current_file.cpp"      修给回实际的cpp文件的名字。

-------------------------



修改后_LINE__   __FILE__  等宏相应的也受影响



visual stuido c++ 里面也同样的东西  #line Directive (C/C++)

http://msdn.microsoft.com/en-us/library/b5w2czay(v=vs.110).aspx



做代码自动生成工具的时候，可以考虑一下使用这个宏。
