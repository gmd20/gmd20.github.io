https://github.com/torvalds/linux/blob/master/Documentation/static-keys.txt

这个	static_branch_likely() 	static_branch_unlikely() 有点像likely和unlikely，但这个其实完全避免了分支指令了，全部插入nop指令了，然后
修改变量的值的时候才插入不同的代码，非常有意思
