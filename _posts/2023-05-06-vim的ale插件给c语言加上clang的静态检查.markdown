https://github.com/dense-analysis/ale  

这儿插件比较简单，widnows上安装 llvm后，自动给C语言加上 lint 语法检查了，golang的也有检查的，这样可以比较一些低级语法错误了。

但LLVM默认的缺少c语言头文件的，按照官方文档 https://clangd.llvm.org/guides/system-headers 
是要自己安装c语言的头文件的。 但我不按照VC，用到了linux环境，自己根据“clang.exe -v -c -xc++ nul” 输出，把Linux 的"/usr/include"
打包出来放到 “C:\Program Files\Microsoft Visual Studio 10.0\VC\include” 这个它默认会搜索目录就行了。

这样ALE默认的错误提示会少很多。
