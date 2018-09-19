```text
    




  下载LOFTER我的照片书  |
编译程序时，发现一个奇怪的错误，明明有 -lz 指定了 zlib的链接库了。就是找不到zlib里面函数。
原来是因为 *.o  *.a写在 -lz的后面了。  这恶 -l选项指定库时是有顺序的。 只对它前面的那些 *.o文件起作用。
参考
https://gcc.gnu.org/onlinedocs/gcc/Link-Options.html
-llibrary
-l library
Search the library named library when linking. (The second alternative with the library as a separate argument is only for POSIX compliance and is not recommended.)
It makes a difference where in the command you write this option; the linker searches and processes libraries and object files in the order they are specified. Thus, ‘foo.o -lz bar.o’ searches library ‘z’ after file foo.o but before bar.o. If bar.o refers to functions in ‘z’, those functions may not be loaded.

The linker searches a standard list of directories for the library, which is actually a file named liblibrary.a. The linker then uses this file as if it had been specified precisely by name.

The directories searched include several standard system directories plus any that you specify with -L.

Normally the files found this way are library files—archive files whose members are object files. The linker handles an archive file by scanning through it for members which define symbols that have so far been referenced but not defined. But if the file that is found is an ordinary object file, it is linked in the usual fashion. The only difference between using an -loption and specifying a file name is that -l surrounds librarywith ‘lib’ and ‘.a’ and searches several directories. 



所以 -l 这个选项最好放到命令的最后面。改成放到*.a之后就编译通过了。


```
