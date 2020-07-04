昨天发现一个程序的bug
```cpp
unsigned long mask = 1 << 1;
for (unsigned long i = 0; i < 64; i++) {
  if (mask & (1 << i)) {
    printf("%lu", i);
  }
}
```
我写这段代码的时候，觉得应该只会输出1，但centos8 + gcc 输出是 1 和 33，一下颠覆了我的认知。
这里的原因就是1这个常量整数时32位的int，然后左移了33位之后就溢出了再次从头开始了,（1 << 33）结果等于( 1 << 1)。
所以代码改成这样才是我想要的结果
```cpp
unsigned long mask = 1;
for (unsigned long i = 0; i < 64; i++) {
  if (mask & (1UL << i)) {
    printf("%lu", i);
  }
}
```

但刚又在 windows visual studio 测试了一下，才发现 visual stuido c++， 64位系统的unsinged long 是4个字节长度的， long long才是8个字节，好久没写Windows代码都忘记了。
有点放人类啊，不知道微软的人怎么想的。看来代码还是用uint64_t这种比较好了。
https://docs.microsoft.com/en-us/cpp/cpp/data-type-ranges?view=vs-2019
https://software.intel.com/content/www/us/en/develop/articles/size-of-long-integer-type-on-different-architecture-and-os.html
