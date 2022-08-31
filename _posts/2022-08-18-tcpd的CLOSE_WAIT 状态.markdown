发现http服务器有一个bug，有一个CLOSE_WAIT状态的socket导致 反复的读取死循环100%，只有重启进程才能修复。

网上找的 https://stackoverflow.com/questions/15912370/how-do-i-remove-a-close-wait-socket-connection
```text
CLOSE_WAIT means that the local end of the connection has received a FIN from the other end, but the OS is waiting for the program at the local end to actually close its connection.

The problem is your program running on the local machine is not closing the socket. It is not a TCP tuning issue. A connection can (and quite correctly) stay in CLOSE_WAIT forever while the program holds the connection open.

Once the local program closes the socket, the OS can send the FIN to the remote end which transitions you to LAST_ACK while you wait for the ACK of the FIN. Once that is received, the connection is finished and drops from the connection table (if your end is in CLOSE_WAIT you do not end up in the TIME_WAIT state).
```

有提到ss -K 命令可以关闭tcp socket，但尝试了一下没有成功
```text
ss --tcp state CLOSE-WAIT --kill
ss --tcp state CLOSE-WAIT '( dport = 22 or dst 1.1.1.1 )' --kill
ss --tcp --kill sport = 54576 or dport = :ssh
```

编程的话，可以通过EPOLLRDHUP 事件判断对端已经关闭socket
```text
       EPOLLRDHUP (since Linux 2.6.17)
              Stream socket peer closed connection, or shut down writing
              half of connection.  (This flag is especially useful for
              writing simple code to detect peer shutdown when using
              edge-triggered monitoring.)
```

lighttpd的代码判断连接是否是 CLOSE_WAIT状态
```c
    struct tcp_info tcpi;
    socklen_t tlen = sizeof(tcpi);/*SOL_TCP == IPPROTO_TCP*/
    return (0 == getsockopt(fd,     SOL_TCP, TCP_INFO, &tcpi, &tlen)
            && tcpi.tcpi_state == TCP_CLOSE_WAIT);
```
