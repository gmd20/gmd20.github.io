参数配置cat /etc/ppp/pppoe-server-options
```text
# PPP options for the PPPoE server
# LIC: GPL
auth
require-pap
require-chap
# login
lcp-echo-interval 10
lcp-echo-failure 2
ms-dns 119.29.29.29
ms-dns 180.76.76.76

```


配置用户名和密码
```
/ # cat /etc/ppp/pap-secrets 
# Secrets for authentication using PAP
# client	server	secret			IP addresses
test *       pwd1 *
/ # cat /etc/ppp/chap-secrets 
# Secrets for authentication using CHAP
# client	server	secret			IP addresses
test *       pwd1 *

```


运行pppoe-server
```text
pppoe-server -I eth2 -L 192.168.1.1 -R  192.168.1.2 -k -N 100
```

然后windows系统里面 “PPPOE拨号”连接就可以连接登录成功了
