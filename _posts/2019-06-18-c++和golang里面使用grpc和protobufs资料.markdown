
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
// Parse a protocol buffer contained in an array of bytes.
bool ParseFromArray(const void* data, int size);
  
bool SerializeToString(string* output) const: Serialize the given message to a binary string.
string DebugString(): Return a string giving the `text_format` representation of the proto (should only be used for debugging)
  // Serialize the message and store it in the given byte array.  All required
  // fields must be set.
  bool SerializeToArray(void* data, int size) const;
  // Like SerializeToString(), but appends to the data to the string's
  // existing contents.  All required fields must be set.
  bool AppendToString(std::string* output) const;
  
  // Computes the serialized size of the message.  This recursively calls
  // ByteSizeLong() on all embedded messages.
  //
  // ByteSizeLong() is generally linear in the number of fields defined for the
  // proto.
  virtual size_t ByteSizeLong() const = 0;
  
  常用的Message消息的接口序列化和反序列化函数就上面那些了，自动生成的代码还给每个结构的子元素生成了set_xxxx的赋值函数。
  
  注意 c++ 有两个库可以链接， libprotobuf.a  libprotobuf-lite.a， lite的版本对应嵌入式比如Android等设备里面用，
  提供的接口没有那么多，但代码会比较小一些。 要用lite版本的库，需要在 proto文件里面指定 “option optimize_for = LITE_RUNTIME;”
  然后再生成代码就可以了

  https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/message_lite.h
  https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/message.h
  
  
```


# grpc的接口设计和命名规范
```text
https://cloud.google.com/apis/design/errors
https://developers.google.com/protocol-buffers/docs/style
```
