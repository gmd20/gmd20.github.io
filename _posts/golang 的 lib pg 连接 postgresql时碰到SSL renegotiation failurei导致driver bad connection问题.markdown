1. 使用 golang  lib/pg https://github.com/lib/pq 去连接postgresql时候发生这个错误
-----------------
```
2015/07/27 17:48:30 main.go:392: driver: bad connection
```

出错的代码， rows.Err() 里面得到的错误
```go
  for rows.Next() {
    err = rows.Scan(&id, &typ, &status)
  }

  err = rows.Err() // get any error encountered during iteration
  if err != nil {
    log.Fatal(err)
  }
```

2.  查看postgresql的log，可以看到连接确实断开了
---------------
```
2015-07-27 17:48:30 CST [21997-51] user1@user1 LOG:  SSL renegotiation failure
2015-07-27 17:48:30 CST [21997-52] user1@user1 STATEMENT:  SELECT id,typ,status,
2015-07-27 17:48:30 CST [21997-53] user1@user1 LOG:  SSL error: ssl handshake failure
2015-07-27 17:48:30 CST [21997-54] user1@user1 STATEMENT:  SELECT id,typ,status,
2015-07-27 17:48:30 CST [21997-55] user1@user1 LOG:  could not send data to client: Connection reset by peer
2015-07-27 17:48:30 CST [21997-56] user1@user1 STATEMENT:  SELECT id,typ,status,
2015-07-27 17:48:30 CST [21997-57] user1@user1 FATAL:  connection to client lost
2015-07-27 17:48:30 CST [21997-58] user1@user1 STATEMENT:  SELECT id,typ,status,
```

看上去是  SSL renegotiation failure 的错误




3.  在网上找到一个官方的issue，说是golang 暂时还是不支持"renegotiation"的
---------------------------
crypto/tls: does not support renegotiation #5742
https://github.com/golang/go/issues/5742

“TLS renegotiation" 的client端的实现不支持这个功能，因为 “IETF RFC” 都没解决这个导致"triple-handshake attacks[1]" 的问题

参考这个链接
https://www.secure-resumption.com/
Triple Handshakes Considered Harmful
Breaking and Fixing Authentication over TLS

暂时没看到官方有修复的说法。

只能找其他办法了。


4. 在postgresql里面可以关闭这个选项
-----------------------------------
```
ssl_renegotiation_limit (integer)
Specifies how much data can flow over an SSL-encrypted connection before renegotiation of the session keys will take place. Renegotiation decreases an attacker's chances of doing cryptanalysis when large amounts of traffic can be examined, but it also carries a large performance penalty. The sum of sent and received traffic is used to check the limit. If this parameter is set to 0, renegotiation is disabled. The default is 512MB.

Note: SSL libraries from before November 2009 are insecure when using SSL renegotiation, due to a vulnerability in the SSL protocol. As a stop-gap fix for this vulnerability, some vendors shipped SSL libraries incapable of doing renegotiation. If any such libraries are in use on the client or server, SSL renegotiation should be disabled.
```

默认的限制是 512M，怪不的流量越大越快重现这个问题。 改成 0 ，重启postgresql 应该就解决问题了。
