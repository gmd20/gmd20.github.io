top或者free命令发现linux的page cache文件缓存占用的内存特别多，但这些命令没法都看到是那些文件被缓存的，在进程的内存信息里面也没看的出来。

在网上找了一下，提到几个命令吧

fincore
=======
```text
# rpm -qf /usr/bin/fincore
util-linux-2.32.1-22.el8.x86_64
```
centos 8 自带的一个命令，但好像不太好用
```text
# fincore  1.txt
  RES PAGES  SIZE FILE
21.7M  5555 21.7M 1.txt
```
需要你指定文件名，然后他就告诉你这个文件被缓存了多大的吧，不过也支持传多个文件名的，和通配符的写法也还可以吧。有人还写脚本lsof把结果给fincore来调用。
```text
 fincore ./*
```


pcstat
======
https://github.com/tobert/pcstat
这个好像说是fincore类似


vmtouch
=======
https://github.com/hoytech/vmtouch
这个支持目录的写法，使用比较简单一些，最后找出来是 日志文件太大了，因为日志文件一直打开着不停的写，所有文件缓存没有被系统page cache后台服务器所回收。


网上的文章
=========
有的提到用systemtap直接遍历内存cache信息来统计的，可以改写成 kernel module的方式吧，不过杀鸡用牛刀的感觉。
上面的几个命令应该够用了，无论是查看某个文件使用的缓存有多大，还是排查缓存都被哪些文件占用了。

https://blog.csdn.net/rapheler/article/details/52528577
http://blog.yufeng.info/archives/688

