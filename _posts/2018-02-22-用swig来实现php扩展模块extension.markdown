看了一下，使用swig实现php扩展模块比直接用c来写应简单很多。swig自动生成包装函数简化了很多工作。


1. 下载swig回来自己编译一下

2.  自己实现*.c 源码文件。

3. 定义 swig的接口  *.i
```text
%module my_extension

%pragma(php) phpinfo="
  zend_printf(\"my PHP extension\\n\");
  php_info_print_table_start();
  php_info_print_table_header(2, \"INFO\", \"Value\");
  php_info_print_table_row(2, \"version\", \"1.0\");
  php_info_print_table_end();
"

%minit {
  my_module_init();
}

%mshutdown {
  my_module_shutdown();
}

extern int xxxx_set_value(unsigned int param1, char param2[64]);

```
非常简单，  xxxx_set_value 函数就是自己在c文件中实现的要导出给php使用的函数名字。 %minit 是php模块初始化时调用的代码，参考PHP和swig的文档。

4. 用swig生成接口和编译
```text
#!/bin/bash

../swig-3.0.12/swig -I../swig-3.0.12/Lib/ -I../swig-3.0.12/Lib/php5  -php5 example.i
gcc -I/home/php-5.6.32 -I/home/php-5.6.32/Zend -I/home/php-5.6.32/TSRM -I/home/php-5.6.32/main  -fpic -c example_wrap.c example.c
gcc -shared -Wl,-rpath=/usr/mylib/lib -lmylib -L/home/mylib example_wrap.o example.o -o example.so

```
swig 会根据  example.i 生成 example_wrap.c 文件，这个就是php接口的包装函数。
后面的编译就跟普通的应用有什么区别了。


#参考
```text
http://www.swig.org/Doc3.0/Php.html
http://php.net/manual/en/internals2.structure.lifecycle.php
https://devzone.zend.com/303/extension-writing-part-i-introduction-to-php-and-zend/
https://www.ibm.com/developerworks/cn/opensource/os-php-swig/index.html
```
