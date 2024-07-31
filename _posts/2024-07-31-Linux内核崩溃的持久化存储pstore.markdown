看到有个 /sys//fs/pstore 文件系统，找了一下，原来是把日志 存储到 内存的东西，类似kdump的原理。不过这个还支持blk 把崩溃日志写到 某磁盘分区的。
```text
persistent storage file system 

fs/pstore/pstore



mount -t pstore pstore /sys//fs/pstore


[root@localhost ]# find /root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/

/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/Kconfig
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/Makefile
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/blk.c
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/ftrace.c
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/inode.c
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/internal.h
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/platform.c
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/pmsg.c
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/ram.c
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/ram_core.c
/root/rpmbuild//BUILD/kernel-5.14.0-427.24.1.el9_4/linux-5.14.0-427.24.1.el9.x86_64/fs/pstore/zone.c

ramoops
[root@localhost  ]# modinfo ramoops
filename:       /lib/modules/5.14.0-427.24.1.el9_4.x86_64/kernel/fs/pstore/ramoops.ko.xz

https://kernel.org/doc/html/latest/admin-guide/ramoops.html
https://elixir.bootlin.com/linux/v6.10.2/source/Documentation/admin-guide/ramoops.rst
https://docs.kernel.org/admin-guide/pstore-blk.html
https://www.cnblogs.com/dongxb/p/18011157
https://www.ais.com/understanding-pstore-linux-kernel-persistent-storage-file-system/







systemd的 man pstore.conf
```
