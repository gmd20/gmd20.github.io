以前的常用的编译期错误报告，就是#if  和 #error 的宏检查吧，不过这个能针对宏的值做检查，
c语言有一个很古老的assert函数，这个运行时检查会影响性能吧。
不过c11/c++11 加了一个 static_assert，这个的好处就是支持对所有常量的判断，而且是编译期的检查不影响性能。

印象中记得有这么一个东西，上次想在内核里面用的时候没找到，刚看了一下内核代码发现linux内核已经包装了 BUILD_BUG_ON系列的宏了，不过static_assert应该也还能用。
https://elixir.bootlin.com/linux/latest/source/include/linux/build_bug.h

