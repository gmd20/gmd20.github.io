才发现别人的c++开源项目里面也用了nuget来配置依赖的外围库。
之前的安装visutal studio 发现nuget在program file 目录下载了很多.NET的包。

这个c++也用nuget还是第一见到。 可能是跟nodejs还有java之类的学的吧，有统一的包管理平台，各个开源软件发布起来确实方便了。
开发者也只需要设置好对应的依赖关系，编译的时候就会自动下载这些依赖的库了，不需要自己再编译。

“解决方案视图” 里面右键 项目名字，选 “nuget包管理器”就可以进行设置了。找到自己要的库添加就可以了。

看了一下，visual studio 会把下载下来的库放到packages 目录里面。c++ 只要设置好include和lib指向这个目录就可以了。
比如这种 packages\libiconv.1.14.0.11\build\native\include

nuget的官方文档 https://docs.microsoft.com/zh-cn/nuget/




