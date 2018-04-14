
参考内核文档 “6. Module Versioning”
https://www.kernel.org/doc/Documentation/kbuild/modules.txt


当编译内核时配置了CONFIG_MODVERSIONS选项， 会根据内在编译的MODPOST阶段根据内核源码为每个模块的导出函数   
生存一个CRC校验。 全部的模块的函数的这个CRC版本信息在make modules后会保存在源码的根目录的Module.symvers    
文件里面。 单独编译其他模块依赖其他模块的符号时会使用这个文件，加入版本信息，如果找不到Module.symvers 编译时   
会打印警告的。   

Module.symvers file 的格式是这样:   
```text
		<CRC>	    <Symbol>	       <module>   
```
    

内核加载模块是会检查这个CRC版本信息，如果两个模块的导出方和导入方的版本不一致，也就提示内核接口变化了。 会拒接   
加载，会打印 “ disagrees about version of symbol” 这样的信息。 如果编译时缺失Module.symvers 导致编译出来   
的模块里面没有携带版本信息也是会报格式错误的。

