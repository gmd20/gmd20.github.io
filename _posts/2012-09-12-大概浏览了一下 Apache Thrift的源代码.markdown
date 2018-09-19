    




  下载LOFTER我的照片书  |
代码很容易看，结构设计的很好啊！

参见http://thrift.apache.org/docs/concepts/  的说明，抽象出 server  Processor Protocol Transport四个 接口后，代码的层次分明，读起来很容易，确实值得学习啊！  然后再这这个基本接口上面，继承实现各种模型。



processor :     StatsProcessor.h  PeekProcessor.h   TAsyncProcessor

protocol  :      TBase64Utils.h   TBinaryProtocol.h TCompactProtocol.h TJSONProtocol.h

server :          TNonblockingServer.h  TSimpleServer.h    TThreadPoolServer.h    TThreadedServer.h

transport :     TBufferTransports.h  TFileTransport.h THttpServer.h THttpClient.h TPipe.h TSSLSocket.h TSocket.h TZlibTransport.h



想message的 打包，解包的实现，实在 protocol里面去做的，都用统一的接口，像 BinaryProtocol 的代码简单明了，又有高性能！ 佩服啊！



idl的compiler部分，词法解析部分使用的flex和bison两个库，没看懂。值的学习一下，以后写解析器类的东西，可以考虑使用这两个 库来做。

flex

http://flex.sourceforge.net/ 

Bison

http://www.gnu.org/software/bison/ 
