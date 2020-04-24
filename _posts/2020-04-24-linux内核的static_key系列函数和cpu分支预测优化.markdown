https://github.com/torvalds/linux/blob/master/Documentation/static-keys.txt

这个	static_branch_likely() 	static_branch_unlikely() 有点像likely和unlikely，但不像likely这个其实完全避免了分支指令了，全部插入nop指令了，然后修改变量的值的时候才把nop替换为跳转到不同分支的指令，非常有意思。 之前看有一个说内核的ftrace就是用类似的技术实现的吧。
