squashfs 是压缩只读文件系统，类似的文件系统有很多，但这个好像比较流行一些吧。
overlayfs 把另外一个可写的目录和这个只读目录合并起来实现“copy-on-write”机制的可写目录。

这两个都已经被收入最新的linux内核里面去了。
这个好像是一个比较流行的组合， squashfs提供高压缩比的只读镜像，然后overlayfs在squashfs上面提供可写功能。

一些嵌入式发行版比如openwrt会用，还有ubuntu等的liveCD也就是系统安装光盘，还有Docker的镜像系统都会用
这两个文件系统来实现。



检查系统是否支持overlay和squashfs文件系统
--------------------------------------------
```bash
modprobe overlay
modprobe squashfs
cat /proc/filesystems

```

制作和解压squashfs文件系统相关的命令
------------------------------------
```
mksquashfs
unsquashfs

```

参考资料 
--------
[The Overlay Filesystem](http://windsock.io/the-overlay-filesystem/)

[内核帮助文档overlayfs.txt](https://www.kernel.org/doc/Documentation/filesystems/overlayfs.txt)
[内核帮助文档squashfs.txt](https://www.kernel.org/doc/Documentation/filesystems/squashfs.txt)

[wiki OverlayFS](https://en.wikipedia.org/wiki/OverlayFS)
[wiki SquashFS](https://en.wikipedia.org/wiki/SquashFS)

