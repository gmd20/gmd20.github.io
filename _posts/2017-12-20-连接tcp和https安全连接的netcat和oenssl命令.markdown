之前之知道有netcat，不知道这个 openssl s_client 还可以用来直接连ssl的socket。
```text
telnet 163mx01.mxmail.netease.com 25
netcat -C  163mx01.mxmail.netease.com 25
openssl s_client -crlf -connect cn.bing.com:443 <<EOF
GET / HTTP/1.1
Host: cn.bing.com
EOF
```
