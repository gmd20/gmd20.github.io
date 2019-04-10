```text
ssh -nNT -L 8000:localhost:3306 user@server.com
```
在本地连接8000端口，就等价于连接 远程server机器上的 localhost:3306  端口

```text
ssh -nNT -R 4000:localhost:3000 user@server.com
```
反过来，在远程server上面连接127.0.0.1:4000端口就等价于 本地机器的localhost:3000端口

```text
ssh -D 5000 -nNT user@server.com
```
SOCKS5 SOCKS4浏览器代理，把本地 localhost:5000 设置为浏览器代理， 流量就从远程server上面转发出去了

```text
ssh -nNT -R 0.0.0.0:4000:192.168.1.101:631 user@server.com
```
在远程机器上面开一个0.0.0.0:4000监听端口，本地的192.168.1.101:631 端口

原文
https://medium.com/tarkalabs/power-of-ssh-tunneling-cf82bc56da67
