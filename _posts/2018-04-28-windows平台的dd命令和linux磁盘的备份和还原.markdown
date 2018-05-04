linux下面的dd命令可以直接读取原始磁盘扇区进行备份和还原，比如这样
```sh
dd if=/dev/sdb  of=sdb.mbr bs=512 count=2048
dd if=/dev/sdb1  of=sdb1.img 
dd if=sdb1.img of=/dev/sdb1
```

windows下面安装了git for windows之后，git bash自带的msys mingw里面其实是由dd命令的。   
而且msys也对/dev/sda  /dev/sdb1 这样的路径做了虚拟和map的。   
就是说同样的dd命令在windows平台也是一样可以运行的, 测试一下是可以成功的。但注意的一点就是 “/dev/sdb” 这样的   
磁盘设备而不是/dev/sdb1这样分区进行读取的时候，*需要管理员权限才能访问*，才开始菜单搜索git bash，右键选择“以管理员身份运行”，   
然后再进行操作就行了。   
 
其实本来windows应该是支持  '\\.\physicaldrive0' 这样的来打开物理设备  '\\.\C:' 这样来打开逻辑设备的进行操作的， 
有管理员权限，使用CreateFile打开这种设备文件，直接读写文件就直接读取磁盘扇区的内容了“direct disk access (raw I/O) ”
参考：
Direct Drive Access Under Win32   
https://support.microsoft.com/en-us/help/100027/info-direct-drive-access-under-win32   


rufus
=====
如果没有安装git for windows，这个开源的usb启动盘制作软件rufus也是支持把“dd image”类型的精心的，应该是可以用来还原磁盘的。
https://github.com/pbatard/rufus



有写u盘或者读卡器的linux下面驱动支持的不好，就可以在windows上面用dd命令来制作启动盘了，现在linux下面制作dd备份，然后拿到windows来用dd写磁盘。


dd的磁盘复制的问题
=================
1. dd的bs要设置的大一些，比如64k，读写才会快一些。但这个bs应该是会影响读取时的错误处理的，比如磁盘不是64k大小对齐dd是怎么出来错误的？这个要看dd的文档才行。
2. dd的是完全的磁盘原始数据的拷贝，但这样会浪费空间。比如你有一个ext4的分区是100GB的，里面只保存了1MB的数据，那么用dd来备份的话，也是要复制在整个100GB的磁盘来做备份的。虽然可以使用压缩文件，但效果也不一定好，效率也低。这种情况就需要更高级的磁盘备份命令了，比如e2image。


e2image
=======
e2image备份的磁盘镜像会分析ext2 ext3 ext4这些文件系统，只备份磁盘被使用了的扇区。这个如果是比较空闲的磁盘，备份出来的文件应该会比dd小的多吧。
e2image可以保证备份的尽可能小又可以保证文件系统不被破坏。 

我的一个使用场景，就是要在linux上面用dd备份grub和分区表，用e2image备份分区内容，然后在windows上面还原磁盘分区和内容。

windows上面的e2image
====================
[git for windows](https://github.com/git-for-windows) 是居于[msys2](https://github.com/msys2) 和[mingw-w64](https://mingw-w64.org/)来开发的， 最近他们也放出来了[git for  windows 的编译环境](https://github.com/git-for-windows/git-sdk-64),
这个mingw带了windows下面的gcc和gdb等软件了，但找了一下没发现e2image。 尝试了一下在mingw下自己编译e2fsprogs，发现有很多地方修改。

看到有网友提到cygwin下面的dd，试了一下cygwin也是由e2fsprogs包可以直接安装的。这样windows和linux的跨系统的磁盘备份和还原应该就可以执行了。
不过我没有实际测试，按理来说e2image在windows上面直接操作/dev/sda这样的块设备文件应该是可以，而且cygwin的这个包里面还有mkfs.ext4，不过我没测试。
因为现在linux上面的 u盘接口的读卡器驱动有个work around，应用后可以直接在linux下面访问u盘了。

唯一可能担心的问题就是这个mkfs.ext4好像有些版本不一样，创建的ext4文件好像有些特性不太一样，fsck不同版本工作不太正常。连centos和ubuntu上面都观察到ext4这个特性差异造成的fsck无法进行的错误。不知道不同版本的e2image能否兼容。



cygwin和mingw的差异
==================
mingw应该是直接修改代码，使用windows的API来替换原有代码的linux API了。
cygwin和mingw的差异应该是cygwin会自己实现一个posix API的dll，这样linux应用移植过来windows很多源码就不需要修改了。
在mingw编译linux程序需要自己修复linux posix api接口为windows api接口，但在cygwin下面就不要了，因为cygwin虚拟了一个posix API接口给你，
这样工作量应该会小很多。这就难怪有人在cygwin编译e2fsprogs包，但没人在mingw下面编译e2fsprogs包了。
cygwin本身的gcc和gdb等包也是使用的mingw版本的。

