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
