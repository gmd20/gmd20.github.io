dlopen 支持_init函数的   
https://man7.org/linux/man-pages/man3/dlopen.3.html  

不过现在建议使用gcc的constructor attribute,像下面这样定义，xxxx_init 会在初始化阶段dlopen返回之前被调用，和main入口之前被调用吧   
destructor 函数
```c
void __attribute__((constructor)) xxxx_init(void);
void __attribute__((destructor)) xxxx_exit(void);

void xxxx_init(void)
{
}
void xxxx_exit(void)
{
}

```

gcc的文档说明
```text
constructor
destructor
constructor (priority)
destructor (priority)
The constructor attribute causes the function to be called automatically before execution enters main (). Similarly, the destructor attribute causes the function to be called automatically after main () has completed or exit () has been called. Functions with these attributes are useful for initializing data that will be used implicitly during the execution of the program.
You may provide an optional integer priority to control the order in which constructor and destructor functions are run. A constructor with a smaller priority number runs before a constructor with a larger priority number; the opposite relationship holds for destructors. So, if you have a constructor that allocates a resource and a destructor that deallocates the same resource, both functions typically have the same priority. The priorities for constructor and destructor functions are the same as those specified for namespace-scope C++ objects (see C++ Attributes).
```
