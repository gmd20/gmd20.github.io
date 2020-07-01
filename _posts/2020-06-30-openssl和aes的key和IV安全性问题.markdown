# AES算法和模式的选择
 openssl enc -ciphers
 不能用ecb模式，ecb是不安全的。 如果没有特殊要求， -aes-256-cbc  aes-256-cfb 和-chacha20 都可以吧
 
# AES加密算法的CBC模式等要求IV要足够是随机的不可预测的，每次加密都不要用一样的IV， 这样才是安全的

# openssl 不要使用-K 和-iv 使用固定IV， 这样IV就没有随机性不安全了

# openssl 的key和IV的生成
如果不是用-K和-iv指定的话，openssl会自动生成一个随机的salt，然后使用hash算法来生成随机的 key 和IV，这样生成出来的key 和IV每次都是不一样的足够安全的。
salt会保存在加密后的文件头里面。  hash 算法可以用过 -md参数指定，老版本是用md5， 1.1版本用的sha256了吧，所以1.1和老版本openssl可能不指定-md可能不兼容了。
-p -P会打印 key 和IV出来吧

# openssl 使用的例子
```text
openssl enc -k mypassword -md sha512 -iter 256 -pbkdf2 -aes-256-cfb -in 1.txt -out 2.txt
openssl enc -d -k mypassword -md sha512 -iter 256 -pbkdf2 -aes-256-cbc -in 2.txt -out 1.txt 
```
