
1. 生成根证书的私钥
```text
选的椭圆曲线算法，后面生成server和client端的时候类似
$ openssl ecparam -genkey -name prime256v1 -out root_ca.key

也可以选RSA的吧，向下面这样
$ openssl genrsa -out ca.key 2048
```


2. 生成根证书x509格式（公钥）
```text
$ openssl req -new -x509 -days 40000 -key root_ca.key -out root_ca.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:GuangDong
Locality Name (eg, city) []:GuangZhou
Organization Name (eg, company) [Internet Widgits Pty Ltd]:gmd20
Organizational Unit Name (eg, section) []:GZ
