
1.  生成公钥和私钥对
===================
```text
ssh-keygen -t rsa -b 4096 -f <key_file_name> -C "your_email@example.com"
```

会在当前目录生成2个文件
key_file_name   对应私钥
key_file_name.pub   对应公钥

会提示输入一个秘密用来加密私钥，以后每次使用私钥时要输入，这样更安全吧。
也可以不设置秘密这样更方便一些，但别人如果偷走这个私钥文件就可以随意访问了。

```text
$ ssh-keygen.exe  -t rsa -f widebright  -C "bright"   
Generating public/private rsa key pair.   
Enter passphrase (empty for no passphrase):   
Enter same passphrase again:   
Your identification has been saved in widebright.  
Your public key has been saved in widebright.pub.  
The key fingerprint is:   
SHA256:HSf2csIN228fU8pdUijpEYTMnUfIPcEAQZ19W+BNJ40 bright  
The key's randomart image is:   
+---[RSA 2048]----+   
|         ++O+X==o|   
|          + B+E==|  
|          = =.oo=|   
|         + @ o o |   
|        S * * . o|   
|           + o +o|   
|              =o.|   
|             . .o|   
|                .|  
+----[SHA256]-----+  
```
 


https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E7%94%9F%E6%88%90-SSH-%E5%85%AC%E9%92%A5
https://help.github.com/articles/generating-ssh-keys/


2.公钥文件的使用
===============
如果是github这些网站使用, 界面上有提供配置 公钥对页面.
https://help.github.com/articles/generating-ssh-keys/

如果是配置ssh免密码登录, 把公钥文件 key_file_name.pub 文件内容复制到
服务器上面该用户的  ~/.ssh/authorized_keys  文件里面. 注意这文件的权限
,按网上的说法只有该用户才有写权限才行,不能其他用户也可以写
:~$ ls -la .ssh/authorized_keys
-rw-r--r-- 1 bright bright 406 Sep 15 06:45 .ssh/authorized_keys


git的配置, 把公钥加到git用户对应的文件就可以
/home/git/.ssh/authorized_keys

其他 git server像gitosis 这种应该都有自己的配置文档吧
https://github.com/res0nat0r/gitosis



3. 私钥文件的使用
================
git 自带的bash发现他使用的是这个配置文件
全局的配置文件是 /etc/ssh/ssh_config   对应 C:\Git\etc\ssh\ssh_config

针对每个用户的配置文件是  ~/.ssh/config    对应的  C:\Users\<你的真实用户名>\.ssh\config

修改任何一个都会起作用吧。
不过最好修噶 ~/.ssh/config 这个


参考配置文件的说明,  增加对应服务器访问时要用的私钥。
http://linux.die.net/man/5/ssh_config


Host 可以是 域名，或者正则匹配都行
IdentityFile 指定私钥文件

 ```text
Host 192.168.4.3   
  Hostname 192.168.4.3   
  User  <ssh登录名,  这里git server用时一般为git>   
  IdentityFile ~/.ssh/all_sources_code_git   
```
私钥文件的 ~/.ssh/all_sources_code_git 的权限也要正确
	
```text
chmod 700 /root/.ssh/all_sources_code_git

root@debian01:/home/bright/smsc# ls -la /root/.ssh/
总用量 20
drwx------  2 root root 4096  9月 17 18:35 .
drwx------ 13 root root 4096  9月 17 18:35 ..
-rwx------  1 root root 1766  9月 14 16:53 all_sources_code_git
-rw-r--r--  1 root root 1671  9月 17 18:35 config
-rw-r--r--  1 root root 3102  9月 17 18:42 known_hosts
```	


修改完可以   ssh 192.168.4.3  这样测试一下,
看看是不是能够读入配置文件的用户名这些来登录了.

ssh git@192.168.4.3
git  pull
如果一些正常, 配置完之后,
这些命令应该就不需要密码就可以登录到服务器了. 如果一开始ssh-keygen
那一步给私钥设置了密码.  会提示输入 私钥的密码


xmanager/xshell 等工具里面,也可以在认证设置里面选择 才用public key方式,
         然后倒入ssh key 使用



4 TortoiseGit 要使用的putty key 格式私钥要用它自带的puttygen 工具转换一下。
=========================================================
C:\Program Files\TortoiseGit\bin\puttygen.exe

导入 ssh-keygen生成的私钥，另存为 puutygen 的 ppk格式文件。

然后在 TortoiseGit 里面 setting-> Git -> Remote  -> origin

URL 填入git 协议的地址 git@192.168.4.3:project_URL
putty key 制定, 刚才导出的私钥文件就可以了.



5.  ssh-agent  ssh-add 帮助管理私钥的密码，免得每次输入私钥的密码
================================================================
github有文档
https://help.github.com/articles/working-with-ssh-key-passphrases/
