用systemtap跟踪打印动态链接库的所有c++函数调用过程
=================================================

1. ltrace 的问题
---------------
用ltrace 可以打印所有的so文件调用了。但上次试过如果so是自己用dlopen来加载的。
就是在elf结构里面没有对应的依赖项的情况下，ltrace好像是没有做对应的调用了。
用systemtap的就没有这个问题，打印输出也更灵活一些。

2. systemtap的解析c++ 函数问题
------------------------------
systemtap的 probe point 指定函数名，是支持c++ 方式指定名字的。
但默认的打印输出却只能输出函数名，不包含c++ 类名。

c++ 的函数符号是经过修饰的，比如用 nm *.so 查看就可以看到 _Z开头一大堆函数名字。

# c++filt _ZZN9log4cplus19initializeLog4cplusEvE11initialized
log4cplus::initializeLog4cplus()::initialized

systemtap的 ppfunc ()  函数只输出"initialized"
systemtap的 probefunc ()  函数输出 "_ZZN9log4cplus19initializeLog4cplusEvE11initialized"
其实我们想要的是 “log4cplus::initializeLog4cplus()::initialized”
但看一下文件是没有这个tapset函数的，估计的自己写一个才行。

还好有c++filt  这个命令，可以做c++ 函数名的demangle。 我们就可以简单用probefunc 打印
然后在自己用c++filt 命令来转换一下好了。


3. systemtap的函数调用跟踪
---------------------------
systemtap的资料就有一个 para-callgraph.stp  ，根据自己的要求做修改吧。

我是这样用的，打印所有libTest.so 源码文件 *.hpp  *.cpp 的所有函数，
也就是只有c++的函数。 其实用来跟踪c函数调用那些用起来更好用吧，都不用做c++
函数名转换了。根据自己要求来设置跟踪哪些函数吧。
 stap para-callgraph.stp   'process("/usr/lib/libTest.so").function("*@*pp")'  'process("/usr/lib/libTest.so").function("Init")' > trace_file.txt


```stap
#! /usr/bin/env stap

function trace(entry_p, extra) {
  /* %( $# > 1 %? if (tid() in trace) %) */
  %( $# > 1 %? if (2 > 1) %)
  printf("%s%s%s %s\n",
         thread_indent (entry_p),
         (entry_p>0?"->":"<-"),
         /* ppfunc (),  只有函数名，没有包含类名*/
         probefunc (),
         extra)
}


%( $# > 1 %?
global trace
probe $2.call {
  trace[tid()] = 1
}
probe $2.return {
  delete trace[tid()]
}
%)

probe $1.call   { trace(1, $$parms) }
probe $1.return { trace(-1, $$return) }

// 第一个参数是跟踪的打印的函数，第二个参数是触发记录的函数。
// https://sourceware.org/systemtap/langref/Probe_points.html#SECTION00051300000000000000
// probe_point = <function name@filename:line_number>
// stap para-callgraph.stp 'kernel.function("*@fs/*.c")' 'kernel.function("sys_read")':
// stap para-callgraph.stp  'process("/usr/lib/libtest.so").function("*")' 'process("/usr/lib/libtest.so").function("TestInit")'
// stap para-callgraph.stp  'process("/usr/lib/libtest.so").function("*@*cpp")' 'process("/usr/lib/libtest.so").function("TestInit")'
// stap para-callgraph.stp  'process("/usr/lib/libtest.so").function("*@*cpp:*")' 'process("/usr/lib/libtest.so").function("TestInit")'
// stap para-callgraph.stp  'process("/usr/lib/libtest.so").function("*@*cpp:1-200")' 'process("/usr/lib/libtest.so").function("TestInit")'

```


4. 写个简单的perl脚本来调用c++filt 来做函数名转换
----------------------------------------------

```stap
#!/usr/bin/perl -w
use strict;


open(INFILE, "trace_file.txt");
open(OUTFILE, ">trace_file_2.txt");

my $line;
while ($line = <INFILE>){
  if ($line =~ /boost|Trace/) {
    next;
  }
  if ($line =~ /(.*)(_Z\S+)(.*)/) {
    my $function_name = `c++filt $2`;

    chomp($function_name);
    print OUTFILE  $1, $function_name, $3, "\n";
  } else {
    print OUTFILE $line;
  }
}

close INFILE;
close OUTFILE;
```
