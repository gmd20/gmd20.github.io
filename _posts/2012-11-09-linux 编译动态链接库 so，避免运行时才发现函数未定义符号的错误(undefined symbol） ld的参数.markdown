    



```text

ldd  查看 elf文件依赖的  so 动态链接库   可以  export LD_LIBRARY_PATH=/path 设置 so文件的路径，
nm   -u    *.so  或者 nm  |grep  U 查看  那些在  动态链接库中的符号。

 "U" The symbol is undefinedundefined的 symbol  这种就是表示 在其他 so动态链接库里面定义的。但是如果你的编译的 是so文件，如果符号不在外部任何so文件里面，默认的配置也不会提示错误。而是编译通过。那个自己忘了定义的符号也在 这  undefined  symbol里面，但是运行时就加载不成功了。

       文档说，这种编译so动态链接库时找不到符号（不在任何外部so文件里面，自己的程序也没有定义）也允许编译通过是有原因的，参见 ld 的man 说明   --allow-shlib-undefined 解释(好像英文版的才完整，中文的man ld不完整 可以直接查看网页 https://sourceware.org/binutils/docs-2.24/ld/Options.html#Options )。就是让你链接时用的一个版本的so，运行时加载用的另外一个版本的so，可能你的加载时的so里面有这个符号，所以就先让你找不到符号也编译通过了。如果是编译exe，这中链接时找不到定义的符号的就直接给你报错了。 so动态链接库就不会报错。其实这种特性应该是比较少用，最好在so链接是碰到这个未找到的符号也是报错的好。
	所以我觉得编译的动态链接库的时候最好加上 --unresolved-symbols=ignore-in-shared-libs  或者  --no-undefined 来检查一下。这样如果是自己的疏忽在 .c 源文件里面忘记的 某函数的定义，，编译的时候就可以提示错误了。
这里有3个参数可以使用--undefined symbols 和 --no-allow-shlib-undefined 参数的作用范围不一样而已，--undefined symbols 针对常规object文件，--no-allow-shlib-undefined针对的是符号在外部的未定义的shared object里面。--unresolved-symbols和--undefined symbols 作用差不多，不过更具体一些。
我们的目的主要是编译一个so动态链接库时，把自己object里面未定义的符号report出来就可以了，用--no-undefined和--unresolved-symbols=ignore-in-shared-libs应该可以的。


ld   的参数，  如果直接用gcc 编译，可以用   -Wl,--no-undefined 这样传过去
gcc -shared -Wl,-soname,libb.so.1,--no-undefined -o libb.so.1.2    objectfile 
gcc -shared -Wl,-soname,libb.so.1,--unresolved-symbols=ignore-in-shared-libs  -o libb.so.1.2   objectfile 
---------------------------------------

--no-undefined
-z defs
Report unresolved symbol references from regular object files. This is done even if the linker is creating a non-symbolic shared library. The switch --[no-]allow-shlib-undefined controls the behaviour for reporting unresolved references found in shared libraries being linked in.

----------------------------
--unresolved-symbols=method
Determine how to handle unresolved symbols. There are four possible values for `method':
`ignore-all'
Do not report any unresolved symbols. 
`report-all'
Report all unresolved symbols. This is the default. 
`ignore-in-object-files'
Report unresolved symbols that are contained in shared libraries, but ignore them if they come from regular object files. 
`ignore-in-shared-libs'
Report unresolved symbols that come from regular object files, but ignore them if they come from shared libraries. This can be useful when creating a dynamic binary and it is known that all the shared libraries that it should be referencing are included on the linker's command line.
The behaviour for shared libraries on their own can also be controlled by the --[no-]allow-shlib-undefined option.

Normally the linker will generate an error message for each reported unresolved symbol but the option --warn-unresolved-symbols can change this to a warning.




-----------------------------
       -static
           Do not link against shared libraries.  This is only meaningful on
           platforms for which shared libraries are supported.  The different
           variants of this option are for compatibility with various systems.
           You may use this option multiple times on the command line: it
           affects library searching for -l options which follow it.  This
           option also implies --unresolved-symbols=report-all.  This option
           can be used with -shared.  Doing so means that a shared library is
           being created but that all of the library's external references
           must be resolved by pulling in entries from static libraries.



--allow-shlib-undefined
--no-allow-shlib-undefined
Allows or disallows undefined symbols in shared libraries. This switch is similar to --no-undefined except that it determines the behaviour when the undefined symbols are in a shared library rather than a regular object file. It does not affect how undefined symbols in regular object files are handled.
The default behaviour is to report errors for any undefined symbols referenced in shared libraries if the linker is being used to create an executable, but to allow them if the linker is being used to create a shared library.

The reasons for allowing undefined symbol references in shared libraries specified at link time are that:

A shared library specified at link time may not be the same as the one that is available at load time, so the symbol might actually be resolvable at load time.
There are some operating systems, eg BeOS and HPPA, where undefined symbols in shared libraries are normal.
The BeOS kernel for example patches shared libraries at load time to select whichever function is most appropriate for the current architecture. This is used, for example, to dynamically select an appropriate memset function.


```
