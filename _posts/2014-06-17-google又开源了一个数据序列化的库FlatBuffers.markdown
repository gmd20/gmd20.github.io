     说是比protocol buffer这些要快，关键一点就是数据赋值和读取就是编解码了，不需要额外的编解码步骤了。
看上去基本的数据类型和复合数据类型非常丰富(structs, tables, vectors,..)  ，看上比protocol buffer要简单的样子，使用起来也会方便一些？
	之前转了好多篇 “数据序列化的”各种新的开源库了， thrift ， protocol buffer，Simple Binary Encoding (SBE)。Simple Binary Encoding那个也是为了减少内存分配使用连续内存这些，不过现在这个FlatBuffers做的更好一些，使用起来应该更灵活。
        这个值得参考学习一下，感觉我们自己一个rpc库，如果在message里面引入不同的version，来使用不同的编解码方式，应该也是可行的。


自己google一下吧，官方的blog被墙了
http://google.github.io/flatbuffers/md__white_paper.html
https://github.com/google/flatbuffers
http://google.github.io/flatbuffers/md__cpp_usage.html


protocol buffer的作者后来又弄了一个Cap'n Proto，他写了一篇文章比较了一下各个库的区别。
Cap'n Proto, FlatBuffers, and SBE
https://kentonv.github.io/capnproto/news/2014-06-17-capnproto-flatbuffers-sbe.html
阅读(1108)| 评论(3)
