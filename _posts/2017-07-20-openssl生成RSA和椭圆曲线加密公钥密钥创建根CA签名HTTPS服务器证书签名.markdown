
自签名证书和私有CA签名的证书的区别 创建自签名证书 创建私有CA 证书类型 证书扩展名
https://my.oschina.net/jsan/blog/517870
https://linux.cn/article-6498-1.html

上面文章提到的格式和文件扩展名
==============================
x509的证书编码格式有两种
- PEM(Privacy-enhanced Electronic Mail) 是明文格式的
  以 -----BEGIN CERTIFICATE-----开头，以-----END CERTIFICATE-----结尾，
  中间是经过base64编码的内容,apache需要的证书就是这类编码的证书
  查看这类证书的信息的命令为：
```text
  openssl x509 -noout -text -in server.pem
```
  其实PEM就是把DER的内容进行了一次base64编码. openssl 默认应该是这种格式吧

- DER 是二进制格式的证书
  查看这类证书的信息的命令为：
```text
  openssl x509 -noout -text -inform der -in server.der
```


- .crt 证书文件 ，可以是DER（二进制）编码的，也可以是PEM（ ASCII (Base64) ）编码的 ，在类unix系统中比较常见
- .cer 也是证书  常见于Windows系统  编码类型同样可以是DER或者PEM的，windows 下有工具可以转换crt到cer
- .csr 证书签名请求  一般是生成请求以后发送给CA，然后CA会给你签名并发回证书
- .key  一般公钥或者密钥都会用这种扩展名，可以是DER编码的或者是PEM编码的
   查看DER编码的（公钥或者密钥）的文件的命令为
```text
   openssl rsa -inform DER  -noout -text -in  xxx.key
```
   查看PEM编码的（公钥或者密钥）的文件的命令为
```text
   openssl rsa -inform PEM   -noout -text -in  xxx.key
```


RSA/ECC公钥私钥的生成
=====================
检查 openssl版本和支持的加密套件

**目标支持加密套件 ECDHE-ECDSA-AES128-GCM-SHA256，这个性能应该会好一些.**       
**下面是最佳组合的推荐列表，要排除不安全的，还有选择性能好的**         
**https://wiki.mozilla.org/Security/Server_Side_TLS**      

```text

# openssl version
OpenSSL 1.0.2n  7 Dec 2017

# openssl  ciphers -v   | grep ECDSA

检查性能
openssl speed rsa2048 ecdsap256

```

1. 生成 根CA（证书授权中心 certificate authority）要用的RSA密钥
```text
  openssl genrsa -out root_ca.key 8192
```
  如果指定-aes128 参数就会设置一个密码来加密这个pem文件，以后要用到这个pem文件的时候都要通过手工输入密码来解密才能用。
  可以保证私钥的安全，但使用麻烦，比如http 服务器用到的这个私钥每次启动时都要手工输入了

2. 如果只是想生成rsa公钥私钥对的，
  使用下面这个命令从上一步获取到的rsa私钥导出rsa公钥文件
```text
  openssl rsa -in rsa_private.key -pubout -out rsa_public_key.pem
```
  有了这个公钥私钥就可以拿去配置SSH免密码登录，RSA加密等应用了。


3. 另外还可以选择生成椭圆曲线加密(ECC)的私钥和公钥
  一般说ECC的运算速度要快安全性更好，可以使用很短的key获的RSA一样的安全性，
  比如256 位 ECC Key 在安全性上等同于3072位 RSA Key。现在的HTTP服务器nginx apache之类的都可以配置
  同时支持RSA和ECC两种加密，在SSL协商时根据客户端支持情况来选择，如果客户端支持就优先使用ECDHE密钥交换。
```text
  openssl ecparam -genkey -name prime256v1  -out root_ca.key
```

4. 密钥文件格式转换
  有的对证书格式有要求，可以使用pkcs8命令来转换，比如下面把私钥输出成pkcs8格式
```text
  openssl pkcs8 -topk8 -in rsa_private.key -out pkcs8_rsa_private.key -nocrypt
```


HTTPS的SSL连接需要的证书生成
============================

server用自己的私钥自签名
------------------------
1. 首先用前面的方法创建一个RSA或者ECC的私钥，保存到server.key文件
```text
   openssl genrsa -out server.key 8192
   或者
   openssl ecparam -genkey -name prime256v1 -out server.key
   
 ECDH椭圆曲线（对比RSA有性能优势，现在应该是主流选择了）: 
   openssl ecparam -list_curves
   openssl ecparam -out ecparam.pem -name prime256v1
   openssl genpkey -paramfile ecparam.pem -out server.key  
```

2. 生成证书签名请求
```text
   openssl req -new -sha256 -key server.key -out server.csr
```
   这里会提示你输入很多证书信息，很多可以省略，一般Common Name跟你的域名或者
   IP相同，但好像看系统内置的根证书也是可以写其他的。手工输入不太方便， 也可以
   直接使用-subj参数像下面这样制定
```text
   openssl req -new -sha256 -key server.key -out server.csr -subj "/C=CN/ST=GuangDong/L=GuangZhou/O=My Company/OU=My Department/CN=www.test.com/CN=test.com/CN=*.test.com/emailAddress=xxx@163.com"
```
  **但在windows平台的git bash下面，MinGW/MSYS看到参数里面有斜杠就自作聪明的转换
  成目录，可以用cmd //c echo "/CN=Name" 这样的命令打印一下转换后的结果就知道了，
  参考下面文章的解释**
  https://stackoverflow.com/questions/31506158/running-openssl-from-a-bash-script-on-windows-subject-does-not-start-with

  **所以都要写把参数改写成这样，注意参数里面斜杠写法更上面的区别，这样经过MSYS转换
  之后传个openssl就跟上面的命令一样了。这个是windows git bash特有问题，linux平台
  不存在这个情况，linux应该继续使用上面的写法。后面都是使用windows的写法了因为
  我是在windows的git bash里面openssl来测试的**
```text
   openssl req -new -sha256 -key server.key -out server.csr -subj "//C=CN\ST=GuangDong\L=GuangZhou\O=My Company\OU=My Department\CN=www.test.com\CN=test.com\CN=*.test.com\emailAddress=xxx@163.com"
```


   查看csr的内容
```text
   openssl req -text -noout -in server.csr
```

   其他网上的例子：
```text
   指定一个域名
   openssl req -new -sha256 -key domain.key -subj "/CN=yoursite.com" > domain.csr
   指定多个域名好像这样才行
   openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:yoursite.com,DNS:www.yoursite.com")) > domain.csr
```


3. 自己用server的私钥来给server.csr签名
```text
   openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
```

4. 应该也可以用一条命令就做到上面第2和第3步两条命令才完成的工作
```text
   openssl req -new -sha256 -x509 -key server.key -days 3650 -out server.crt \
   -subj "//C=CN\ST=GuangDong\L=GuangZhou\O=My Company\OU=My Department\CN=Heath Test Server\emailAddress=xxx@163.com"
```

5. HTTPS对证书的使用
在golang里面像下面可以启动HTTPS安全连接服务器了，nginx等http服务器也是类似的配置吧
```text
	http.ListenAndServeTLS(":443", "server.crt", "server.key", &h)
```

不过你在浏览器打开这个HTTPS的连接会提示证书错误的，因为是证书我们自己签名的，根证书
不在系统内置的可信任证书列表里面。系统或者浏览器内置的那些证书机构，才是可以被系统
认可的。正确的做法是把csr文件拿到那些受系统信任的CA机构去，让那些CA给我们的服务
器csr签名成crt再发回来，但这些CA机构都是收费的。不过现在有了一个免费的的
[Let’s Encrypt](https://letsencrypt.org/)。不过这个有效期很短，要不断重新申请
，所以一般服务器部署的话都要使用自动化脚本[certbot](https://github.com/certbot/certbot)
来帮我们自动申请，好像openssl这些命令都不需要再手工输入了，脚本会自动化的帮你配
置HTTPS的证书和更新授权了。
另外的几个脚本，他们项目会有些介绍可以看一下。
https://github.com/Neilpang/acme.sh
https://github.com/diafygi/acme-tiny




创建根CA，然后用根CA证书给其他server签名
----------------------------------------
CA的证书授权方式，都是一级一级的的树形结构。一个根CA可以包含很多中级CA，
根CA给中级CA授权，中级CA又可以给其他子节点的做授权。可以查看 [Let’s Encrypt](https://letsencrypt.org/)
也是从其他根CA里面分离出来的，然后[Let’s Encrypt] 才又给我们网站服务器做签名，
我们网站的证书就作为 [Let’s Encrypt](https://letsencrypt.org/)的字节点了。

作为测试，可以自己模拟一下CA授权方式，先创建一个“根CA”， 然后用自己创建的CA证书
给子服务器server.csr签名授权。这样的方式有些好处的，这样我们只要把这个跟CA添加
到系统的“受信任根证书颁发机构” 里面去，各个子服务器的证书也都会被系统认可了。
下面就来创建一个：

1.  首先要准备一个openssl的配置文件和一个目录，因为等会要用第3版的证书，可以支持
   DNS域名和IP的，浏览器可以校验证书里面DNS是不是和网站的实际域名一样。
   其中openssl命令要用到 -config openssl.cnf配置文件包含了ca子命令需要的配置，
   需要自己从系统的模板创建一个，不指定会使用系统默认的文件，比如
   D:/Git/mingw64/ssl/openssl.cnf这种, 可以默认的配置出来在它的基础上面修改一下。
   这个文件我只做了最简单的修改
   **注意创建根证书时使用的是
     “-extensions v3_ca”然后引用的 “basicConstraints = CA:true”
   表示创建的是“根证书”，后面的签名server的crt时使用的是
   “-extensions v3_req” 引用的 “basicConstraints = CA:FALSE”，表示不是 “根CA”
   这点非常重要，只有这样创建出来的整个证书链才是完整的，校验才会认为是有效
   的证书。如果创建CA的crt文件时不是“根CA”类型，校验的时候就认为根CA的上面还有
   其他根CA，但实际又没有，就会认为是非法的证书**

```text
serial 文件是openssl管理颁发序号的,index.txt是对newcerts下面新签发的证书的一个
简单索引

mkdir demoCA
cd demoCA
mkdir newcerts
touch index.txt
echo 01 > serial
cd ..
cat openssl.cnf

[req]
req_extensions = v3_req

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = server1.example.com
DNS.2 = mail.example.com
DNS.3 = www.example.com
DNS.4 = www.sub.example.com
DNS.5 = mx.example.com
DNS.6 = support.example.com
IP.1 = 192.168.1.59
IP.2 = 192.168.1.58

[ v3_ca ]
basicConstraints = CA:true

```


2. 生成根CA的私钥
```text
   openssl genrsa -out root_ca.key 8192
   或者
   openssl ecparam -genkey -name prime256v1 -out root_ca.key
   看看你想用那种加密算了
```

3. 创建SHA-256自签名的根CA证书（CA certificate）
```text
   openssl req -new -sha256 -x509 -days 3650 -config openssl.cnf -extensions v3_ca -key root_ca.key -out root_ca.crt \
     -subj "//C=CN\ST=GuangDong\L=GuangZhou\O=My Company\OU=My Department\CN=Heath\emailAddress=xxx@163.com"
```
   注意 -extensions v3_ca 通过和配置文件里面 “basicConstraints = CA:true”
   指定了签名的类型是 “根CA”。完了之后可以用下面这个命令查看生成的crt是不是
   符合要求的, 注意检查里面的类型是不是 “根CA”。
```text
   openssl x509 -text -noout -in root_ca.crt
```

4. 重新生成server.csr, 使用配置文件，包含subjectAltName dns name进去
```text
openssl req -new -sha256 -key server.key -out server.csr -extensions v3_req -config openssl.cnf \
    -subj "//C=CN\ST=GuangDong\L=GuangZhou\O=My Company\OU=My Department\CN=Heath Test Server"
```
  注意subject里面的stateOrProvinceName等属性要和根CA的stateOrProvinceName等属性一样.
  注意这个参数“ -extensions v3_req” 和配置文件里面DNS 和IP的配置。
  完了检查csr文件，确保签名类型是“非根CA”类型，DNS信息是否存在了
  "X509v3 Subject Alternative Name" -> "DNS"
```text
  openssl req -text -noout -in server.csr
```

5. 用根ca.crt 来给server.csr签名，生成server.crt文件
```text
  openssl ca -days 3650 -cert root_ca.crt -keyfile root_ca.key -md sha256 \
    -extensions v3_req -config openssl.cnf -in server.csr -out server.crt
```
openssl在demoCA目录会做简单的索引，这个命令执行后同样的证书不能再生成第二次了，
需要手工的清理一下demoCA目录index.txt文件内容。

6.  检查证书的有效性
```text
$ openssl verify -verbose -CAfile root_ca.crt   server.crt
server.crt: OK
```
如果前面生成的ca.crt 时没有指定“-extensions v3_ca”生成的 ca.crt 不是根证书，就会报告这个错误，因为openssl就找不到根节点了。
```text
$ openssl verify -verbose -CAfile root_ca.crt  server.crt
server.crt: C = CN, ST = GuangDong, O = Heath Company, OU = Heath Root GZ, CN = Heath inc
error 20 at 0 depth lookup:unable to get local issuer certificate
```
现在就可以把新的server.crt 和server.key 拿到HTTP服务器上面部署了


7. 配置windows系统的"受信任根证书颁发机构"
  在客户端电脑上，资源管理器里面 右键 ca.crt 文件， “安装证书” ，
  选择自定义存储，存到“受信任的根证书颁发机构”里面去。这样再访问我们自签发的
  HTTPS网站,浏览器就不会报告证书错误了. 其他linux系统或者Andorid/iOS之类的应该
  都是有类似的方法来添加自定义证书的吧.
  在IE/chrome浏览器都可以打开证书窗口,对添加的的证书进行管理。但好像只有在
  "certmgr.msc"证书管理程序里面才能删掉某些证书。
