
# 下载编译好的protocol buffer编译器, 最新的protoc-3.8.0-linux-x86_64.zip
```text
https://github.com/protocolbuffers/protobuf/releases
```

# 下载grpc的golang代码生成器protoc-gen-go，自动编译放到$GOPATH/bin目录
```text
go get -u github.com/golang/protobuf/proto
go get -u github.com/golang/protobuf/protoc-gen-go

go get -u google.golang.org/grpc
https://packages.grpc.io/
```

# 编译proto定义文件的例子生成grpc代码
```text
go generate google.golang.org/grpc/examples/helloworld/...
```
其实就是执行下面这个命令生成grpc代码
```text
protoc -I helloworld/ helloworld/helloworld.proto --go_out=plugins=grpc:helloworld
protoc --go_out=plugins=grpc:. *.proto
```


# php的插件
```text
https://pecl.php.net
```
好像也有人编译好的rpm包, 搜索安装 grpc  protobuf

# 编译grpc
```text
git clone https://github.com/grpc/grpc
git submodule update --init
make
```

# 生成cpp的protobuf代码, 非grpc的，只是消息编解码
```text
protoc --cpp_out=. ./test.proto
```

# protobuf的例子
```text
https://github.com/protocolbuffers/protobuf/blob/master/examples/
https://developers.google.com/protocol-buffers/docs/cpptutorial
```

# c++的例子参考
```text
https://developers.google.com/protocol-buffers/docs/reference/cpp/
bool ParseFromString(const string& data): Parse the message from the given serialized binary string (also known as wire format).
bool SerializeToString(string* output) const: Serialize the given message to a binary string.
string DebugString(): Return a string giving the `text_format` representation of the proto (should only be used for debugging)
```

# grpc的接口设计和命名规范
```text
https://cloud.google.com/apis/design/errors
https://developers.google.com/protocol-buffers/docs/style
```
