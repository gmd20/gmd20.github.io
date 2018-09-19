经常看到这种用法，在这里又看到一个了
https://github.com/facebook/folly/blob/master/folly/ProducerConsumerQueue.h

最简单的， “new （指定的地址） ” 其实就是 在 指定的地址已经存在的内存上面调用构造函数建新的对象。 有时可以避免动态申请内存，提高性能，例子参考上面文件里面queue里面插入元素的构建。解释可以参考下面这两个。


Placement syntax
http://en.wikipedia.org/wiki/Placement_syntax

"placement new 操作符"
http://www.cppblog.com/jacky2019/archive/2007/04/06/21375.html
