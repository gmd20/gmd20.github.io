vmalloc好像被改造成自动申请huge page了，后面还增加一个vmalloc_no_huge函数，但好像引入了一些问题，
有的开发者还想增加一个module_alloc_huge函数不知道最后会不会增加进去。

https://lwn.net/Articles/892743/   The BPF allocator runs into trouble
