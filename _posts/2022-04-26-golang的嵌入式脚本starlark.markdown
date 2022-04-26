类似python的语法，

主要的文档：
https://github.com/google/starlark-go     
https://github.com/google/starlark-go/blob/master/doc/spec.md   
https://pkg.go.dev/go.starlark.net/starlark    


例子在telegraf里面有：
https://github.com/influxdata/telegraf/tree/master/plugins/processors/starlark   
https://github.com/influxdata/telegraf/blob/master/plugins/common/starlark/metric.go

不过要在starlark脚本里面访问golang的数据结构还是有些麻烦的，要每个数据结构实现starlark的Value接口，可以参考上面telegraf的例子。
这点没有golang  "tex/template" 模板里面可以直接操作 golang数据结构的属性和方法 方便。starlark胜在语法方便，一些内置函数也完善一些吧。


https://github.com/starlight-go/starlight   
自动为golang 数据结构生成适配接口，但好像没有维护了。





