overlayfs 和squashfs tmpfs这些在内核源码的文档里面都有介绍的。


创建squashfs文件系统
======================
```text
yum  install squashfs-tools
mksquashfs /some/dir dir.sqsh

mksquashfs initrd-busybox initrd-busybox.squashfs
mount -t squashfs initrd-busybox.squashfs /mnt/
```

overlay
===========
```text
  mount -t overlay overlay -olowerdir=/lower,upperdir=/upper,\
  workdir=/work /merged

  workdir是删除文件等操作临时使用的

  多层lower目录，最左边的lower1为最顶层目录最先作用的
  mount -t overlay overlay -olowerdir=/lower1:/lower2:/lower3 /merged


  mkdir -p /tmp/tmpfs/
  mount -t tmpfs -o size=256M tmpfs /tmp/tmpfs
  mkdir -p /tmp/tmpfs/lower
  mkdir -p /tmp/tmpfs/upper
  mkdir -p /tmp/tmpfs/work
  mksquashfs /some/dir initrd-busybox.squashfs
  mount -t squashfs initrd-busybox.squashfs /tmp/tmpfs/lower
  mount -t overlay overlay -olowerdir=/tmp/sysroot:/tmp/tmpfs/lower,upperdir=/tmp/tmpfs/upper,workdir=/tmp/tmpfs/work /tmp/test
```
