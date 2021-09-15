线上发现一个奇怪的现象，从wireshark看到，http 服务器只ack响应client端的syn包，client反复重传数据包，但server端只ack 1 回复client的syn包，
后续http server 会继续重传 syn seq 1 ack 1。这样看起来就是 服务器不响应client端的数据包。

想了半天搞懂不那里出了问题，猜测是tcp和syn 连接队列的溢出的问题。

找了一下资料， 确实可能是这个 连接队列的问题，但根据症状还是不太好找到答案的。

下面这篇文章提到这个现象：
“TCP/IP协议中backlog参数”
https://www.cnblogs.com/Orgliny/p/5780796.html

摘录如下： 
```text
SYN queue: 队列长度由 /proc/sys/net/ipv4/tcp_max_syn_backlog 指定
Accept queue: 队列长度由 /proc/sys/net/core/somaxconn 和使用listen函数时传入的参数，二者取最小值。默认为128。在Linux内核2.4.25之前，是写死在代码常量 SOMAXCONN ，在Linux内核2.4.25之后，在配置文件 /proc/sys/net/core/somaxconn 中直接修改

在LISTEN状态，其中 Send-Q 即为Accept queue的最大值，Recv-Q 则表示Accept queue中等待被服务器accept()。

另外客户端connect()返回不代表TCP连接建立成功，有可能此时accept queue 已满，系统会直接丢弃后续ACK请求；客户端误以为连接已建立，开始调用等待至超时；服务器则等待ACK超时，会重传SYN+ACK 给客户端，重传次数受限 net.ipv4.tcp_synack_retries ，默认为5，表示重发5次，每次等待30~40秒，即半连接默认时间大约为180秒，该参数可以在tcp被洪水攻击是临时启用这个参数。

查看SYN queue 溢出
[root@localhost ~]# netstat -s | grep LISTEN
102324 SYNs to LISTEN sockets dropped

查看Accept queue 溢出
[root@localhost ~]# netstat -s | grep TCPBacklogDrop
TCPBacklogDrop: 2334
```

“另外客户端connect()返回不代表TCP连接建立成功，有可能此时accept queue 已满，系统会直接丢弃后续ACK请求；客户端误以为连接已建立，开始调用等待至超时；服务器则等待ACK超时，会重传SYN+ACK 给客户端，重传次数受限 net.ipv4.tcp_synack_retries”
这个应该就是我在抓包看到的现象。   
netstat -s 的数据应该来自 “/proc/net/netstat”，  “ListenDrops” “TCPBacklogDrop” 这几个就是对应的syn、accept队列溢出事件吧

netstat -s里面这连个是对应 accept队列溢出和syn队列溢出
4873325 times the listen queue of a socket overflowed
5874286 SYNs to LISTEN sockets dropped

ss -lntp 可以看到  Recv-Q Send-Q 的限制， Recv-Q是当前backlog队列长度。

好像还有一个  /proc/sys/net/ipv4/tcp_abort_on_overflow  参数是相关的，参见文章：
“Linux-TCP Queue的一些问题” https://www.cnblogs.com/JohnABC/p/6229136.html
