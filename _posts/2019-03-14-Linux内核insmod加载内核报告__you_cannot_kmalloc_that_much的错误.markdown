2.6的内核的kmalloc最大只能申请128kb的数据结构，调整了一个数据结构，结果最后报错了。

这个有点意思，他编译的时候其实就已经发现问题了，他没有编译警告出来，而是直接链接一个 __you_cannot_kmalloc_that_much 的符号（readelf -s 可以看到），等你insmod的时候才报错。
新的3.10的应该没这么敏感了吧，上次特意在3.10 的64bit系统上测试了一下kmalloc可以分配几兆的数据都是可以的了吧。以前这个128k的限制确实太小了。

这种情况可以用 __get_free_pages 或者 vmalloc 来申请空间吧， 下面是内核里面的一个例子
```c
	if (size <= PAGE_SIZE) {
		return kmalloc(size, GFP_KERNEL);
	} else {
		return (struct hlist_head *)
			__get_free_pages(GFP_KERNEL, get_order(size));
	}
  
  	if (size <= PAGE_SIZE)
		kfree(hash);
	else
		free_pages((unsigned long)hash, get_order(size));
    
```
