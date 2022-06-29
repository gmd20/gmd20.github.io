发现 nslookup  www.baidu.com  curl https://www.baidu.com  经常5秒才会返回，
但nslookup其实很快就返回了A类型的ipv4地址了。 自己抓包发现是， 默认会发送 A 和AAAA这两种类型的 dns查询，
AAAA 类型的响应慢或者无响应导致的，可能有些dns代理缓存对 AAAA类型支持不好导致，还挺常见的。

但AAAA类型查询用不上，但却禁止不了，linux下面禁用ipv6这些都没有用。
这个好像是glibc的默认行为，并没有设置可以禁止掉，  /etc/resolv.conf 的选项没有相关的设置, https://man7.org/linux/man-pages/man5/resolv.conf.5.html
提到的一些设置应该没有用的。
```text
              inet6  Sets RES_USE_INET6 in _res.options.  This has the
                     effect of trying an AAAA query before an A query
                     inside the gethostbyname(3) function, and of
                     mapping IPv4 responses in IPv6 "tunneled form" if
                     no AAAA records are found but an A record set
                     exists.  Since glibc 2.25, this option is
                     deprecated; applications should use getaddrinfo(3),
                     rather than gethostbyname(3).
              single-request (since glibc 2.10)
                     Sets RES_SNGLKUP in _res.options.  By default,
                     glibc performs IPv4 and IPv6 lookups in parallel
                     since version 2.9.  Some appliance DNS servers
                     cannot handle these queries properly and make the
                     requests time out.  This option disables the
                     behavior and makes glibc perform the IPv6 and IPv4
                     requests sequentially (at the cost of some slowdown
                     of the resolving process).

              single-request-reopen (since glibc 2.9)
                     Sets RES_SNGLKUPREOP in _res.options.  The resolver
                     uses the same socket for the A and AAAA requests.
                     Some hardware mistakenly sends back only one reply.
                     When that happens the client system will sit and
                     wait for the second reply.  Turning this option on
                     changes this behavior so that if two requests from
                     the same port are not handled caorrectly it will
                     close the socket and open a new one before sending
                     the second request.                     
```

curl -4 https://www.baidu.com  并不能避免 AAAA查询， 比较新的curl可以通过
curl  --doh-url https://dns.alidns.com/dns-query   https://www.baidu.com 绕过dns的超时，不过并不是想要的结果吧
