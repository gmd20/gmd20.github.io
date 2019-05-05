把这个代码加载需要些heap memory profiler 日志的地方

```go
import (
"runtime/pprof"
)

		f, err := os.Create("HeapProfile.txt")
		if err != nil {
			log.Fatal(err)
		}
		pprof.WriteHeapProfile(f)
		f.Close()
```



启动程序后， 运行到这个地方会生成 HeapProfile.txt  文件

```
go>go tool pprof
usage: pprof [options] [binary] <profile source> ...


D:\smsc\go>go tool pprof  -text  -inuse_objects  smsc_cdr.exe  HeapProfile.txt
386123 of 386124 total (  100%)
Dropped 33 nodes (cum <= 1930)
      flat  flat%   sum%        cum   cum%
    229380 59.41% 59.41%     287809 74.54%  main.generateCdr
     76468 19.80% 79.21%      98314 25.46%  github.com/Shopify/sarama.(*produceSet).add
     38404  9.95% 89.16%      38404  9.95%  bytes.makeSlice
     21846  5.66% 94.81%      21846  5.66%  github.com/Shopify/sarama.(*StringEncoder).Encode
     16384  4.24% 99.06%      16384  4.24%  github.com/lib/pq.textDecode
      3641  0.94%   100%       3641  0.94%  database/sql.convertAssign
         0     0%   100%      38404  9.95%  bytes.(*Buffer).WriteString
         0     0%   100%      38404  9.95%  bytes.(*Buffer).grow
         0     0%   100%      16384  4.24%  database/sql.(*Rows).Next
         0     0%   100%       3641  0.94%  database/sql.(*Rows).Scan
         0     0%   100%      38404  9.95%  encoding/json.(*encodeState).marshal
         0     0%   100%      38404  9.95%  encoding/json.(*encodeState).reflectValue
         0     0%   100%      38404  9.95%  encoding/json.(*encodeState).string
         0     0%   100%      38404  9.95%  encoding/json.(*structEncoder).(encoding/json.encode)-fm
         0     0%   100%      38404  9.95%  encoding/json.(*structEncoder).encode
         0     0%   100%      38404  9.95%  encoding/json.Marshal
         0     0%   100%      38404  9.95%  encoding/json.stringEncoder
         0     0%   100%      98314 25.46%  github.com/Shopify/sarama.(*brokerProducer).(github.com/Shopify/sarama.run)-fm
         0     0%   100%      98314 25.46%  github.com/Shopify/sarama.(*brokerProducer).run
         0     0%   100%      98315 25.46%  github.com/Shopify/sarama.withRecover
         0     0%   100%      16384  4.24%  github.com/lib/pq.(*rows).Next
         0     0%   100%      16384  4.24%  github.com/lib/pq.decode
         0     0%   100%     287809 74.54%  main.main
         0     0%   100%     386124   100%  runtime.goexit

```

go tool pprof <your-executable-name>  HeapProfile.txt 
 进入交互模式，可以输入top，  peek 命令那个查看更详细一些

```
D:\smsc\go>go tool pprof  smsc_cdr.exe  HeapProfile.txt
Entering interactive mode (type "help" for commands)
(pprof) top
79.58MB of 79.58MB total (  100%)
Dropped 28 nodes (cum <= 0.40MB)
Showing top 10 nodes out of 30 (cum >= 0.50MB)
      flat  flat%   sum%        cum   cum%
   51.57MB 64.80% 64.80%    51.57MB 64.80%  bytes.makeSlice
   14.85MB 18.66% 83.47%    14.85MB 18.66%  github.com/Shopify/sarama.encode
    5.66MB  7.11% 90.57%     6.66MB  8.36%  github.com/Shopify/sarama.(*produceSet).add
    5.50MB  6.91% 97.49%    58.07MB 72.97%  main.generateCdr
       1MB  1.26% 98.74%        1MB  1.26%  github.com/Shopify/sarama.(*StringEncoder).Encode
    0.50MB  0.63% 99.37%     0.50MB  0.63%  database/sql.convertAssign
    0.50MB  0.63%   100%     0.50MB  0.63%  github.com/lib/pq.textDecode
         0     0%   100%    51.57MB 64.80%  bytes.(*Buffer).WriteString
         0     0%   100%    51.57MB 64.80%  bytes.(*Buffer).grow
         0     0%   100%     0.50MB  0.63%  database/sql.(*Rows).Next




(pprof) peek
79.58MB of 79.58MB total (  100%)
Dropped 28 nodes (cum <= 0.40MB)
----------------------------------------------------------+-------------
      flat  flat%   sum%        cum   cum%   calls calls% + context
----------------------------------------------------------+-------------
                                           51.57MB   100% |   bytes.(*Buffer).grow
   51.57MB 64.80% 64.80%    51.57MB 64.80%                | bytes.makeSlice
----------------------------------------------------------+-------------
                                           14.85MB   100% |   github.com/Shopify/sarama.(*Broker).send
   14.85MB 18.66% 83.47%    14.85MB 18.66%                | github.com/Shopify/sarama.encode
----------------------------------------------------------+-------------
                                            6.66MB   100% |   github.com/Shopify/sarama.(*brokerProducer).run
    5.66MB  7.11% 90.57%     6.66MB  8.36%                | github.com/Shopify/sarama.(*produceSet).add
                                               1MB   100% |   github.com/Shopify/sarama.(*StringEncoder).Encode
----------------------------------------------------------+-------------
                                           58.07MB   100% |   main.main
    5.50MB  6.91% 97.49%    58.07MB 72.97%                | main.generateCdr
                                           51.57MB 98.10% |   encoding/json.Marshal
                                            0.50MB  0.95% |   database/sql.(*Rows).Scan
                                            0.50MB  0.95% |   database/sql.(*Rows).Next
----------------------------------------------------------+-------------
                                               1MB   100% |   github.com/Shopify/sarama.(*produceSet).add
       1MB  1.26% 98.74%        1MB  1.26%                | github.com/Shopify/sarama.(*StringEncoder).Encode
----------------------------------------------------------+-------------
                                            0.50MB   100% |   database/sql.(*Rows).Scan
    0.50MB  0.63% 99.37%     0.50MB  0.63%                | database/sql.convertAssign
----------------------------------------------------------+-------------
                                            0.50MB   100% |   github.com/lib/pq.decode
    0.50MB  0.63%   100%     0.50MB  0.63%                | github.com/lib/pq.textDecode
----------------------------------------------------------+-------------
                                           51.57MB   100% |   encoding/json.(*encodeState).string
         0     0%   100%    51.57MB 64.80%                | bytes.(*Buffer).WriteString
                                           51.57MB   100% |   bytes.(*Buffer).grow
----------------------------------------------------------+-------------
                                           51.57MB   100% |   bytes.(*Buffer).WriteString
         0     0%   100%    51.57MB 64.80%                | bytes.(*Buffer).grow
                                           51.57MB   100% |   bytes.makeSlice

```


# 通过http接口
http://docs.studygolang.com/pkg/net/http/pprof/
```golang
import  "http"
import _ "net/http/pprof"
go func() {
	log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```
就可以通过下面这些命令，直接连接远程http接口来做性能分析了
```text
go tool pprof http://localhost:6060/debug/pprof/heap
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
wget http://localhost:6060/debug/pprof/trace?seconds=5
go tool pprof http://localhost:6060/debug/pprof/mutex
```



