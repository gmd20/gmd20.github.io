1。 下载最新的 busybox-1.26.2， 
makemenuconfig  去掉不需要的功能（直接改.config文件也可以）
make install 会在 busybox的源码_install目录下创建好所有的命令的软连接。

我编译出来的busybox有600多k。
```text
[ming@centos7 initrd-busybox]$ ls -lh bin/busybox 
-rwxr-xr-x. 1 ming ming 646K Apr 17 14:32 bin/busybox
```

2.  基于这个 _install 目录和当前系统一个正在使用initramfs包来制作我们的最终镜像。 
bin 和lib 和内核驱动都使用busybox和原始initramfs就可以了。
etc/passwd  和 etc/group  etc/shadow 3个文件直接复制当前系统的过来修改。
编译出来的busybox 还依赖另外的c库，也复制过来到lib目录里面。
```text
[ming@centos7 initrd-busybox]$ ldd bin/busybox 
	linux-vdso.so.1 =>  (0x00007ffe589f8000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f6e9d797000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f6e9d3d6000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f6e9daa3000)

```

制作完成的initramfs包含如下文件
-----------------------------
```text
[ming@centos7 initrd-busybox]$ tree 
.
├── bin
│   ├── ash -> busybox
│   ├── base64 -> busybox
│   ├── busybox
│   ├── cat -> busybox
│   ├── catv -> busybox
│   ├── chattr -> busybox
│   ├── chgrp -> busybox
│   ├── chmod -> busybox
│   ├── chown -> busybox
│   ├── cp -> busybox
│   ├── cpio -> busybox
│   ├── date -> busybox
│   ├── dd -> busybox
│   ├── df -> busybox
│   ├── dmesg -> busybox
│   ├── dumpkmap -> busybox
│   ├── echo -> busybox
│   ├── ed -> busybox
│   ├── egrep -> busybox
│   ├── false -> busybox
│   ├── fgrep -> busybox
│   ├── fsync -> busybox
│   ├── getopt -> busybox
│   ├── grep -> busybox
│   ├── gunzip -> busybox
│   ├── gzip -> busybox
│   ├── hostname -> busybox
│   ├── iostat -> busybox
│   ├── ipcalc -> busybox
│   ├── kbd_mode -> busybox
│   ├── kill -> busybox
│   ├── ln -> busybox
│   ├── login -> busybox
│   ├── ls -> busybox
│   ├── lsattr -> busybox
│   ├── mkdir -> busybox
│   ├── mknod -> busybox
│   ├── more -> busybox
│   ├── mount -> busybox
│   ├── mountpoint -> busybox
│   ├── mpstat -> busybox
│   ├── mt -> busybox
│   ├── mv -> busybox
│   ├── netstat -> busybox
│   ├── nice -> busybox
│   ├── pidof -> busybox
│   ├── ping -> busybox
│   ├── ping6 -> busybox
│   ├── printenv -> busybox
│   ├── ps -> busybox
│   ├── pwd -> busybox
│   ├── rm -> busybox
│   ├── rmdir -> busybox
│   ├── rpm -> busybox
│   ├── sed -> busybox
│   ├── sh -> busybox
│   ├── sleep -> busybox
│   ├── stat -> busybox
│   ├── stty -> busybox
│   ├── su -> busybox
│   ├── sync -> busybox
│   ├── tar -> busybox
│   ├── touch -> busybox
│   ├── true -> busybox
│   ├── umount -> busybox
│   ├── uname -> busybox
│   ├── usleep -> busybox
│   ├── vi -> busybox
│   ├── watch -> busybox
│   └── zcat -> busybox
├── cfg
├── data
├── dev
├── etc
│   ├── fstab
│   ├── group
│   ├── hostname
│   ├── init.d
│   │   └── rcS
│   ├── inittab
│   ├── mdev.conf
│   ├── passwd
│   └── shadow
├── init
├── lib
│   └── modules
│       └── 3.10.0-514.10.2.el7.x86_64
│           ├── kernel
│           │   ├── arch
│           │   │   └── x86
│           │   │       └── crypto
│           │   │           ├── crc32c-intel.ko
│           │   │           └── crct10dif-pclmul.ko
│           │   ├── crypto
│           │   │   ├── crct10dif_common.ko
│           │   │   └── crct10dif_generic.ko
│           │   ├── drivers
│           │   │   ├── ata
│           │   │   │   ├── ahci.ko
│           │   │   │   ├── ata_generic.ko
│           │   │   │   ├── ata_piix.ko
│           │   │   │   ├── libahci.ko
│           │   │   │   ├── libata.ko
│           │   │   │   └── pata_acpi.ko
│           │   │   ├── block
│           │   │   │   └── virtio_blk.ko
│           │   │   ├── cdrom
│           │   │   │   └── cdrom.ko
│           │   │   ├── char
│           │   │   │   └── virtio_console.ko
│           │   │   ├── firmware
│           │   │   │   └── iscsi_ibft.ko
│           │   │   ├── input
│           │   │   │   └── serio
│           │   │   │       └── serio_raw.ko
│           │   │   ├── md
│           │   │   │   ├── dm-log.ko
│           │   │   │   ├── dm-mirror.ko
│           │   │   │   ├── dm-mod.ko
│           │   │   │   └── dm-region-hash.ko
│           │   │   ├── net
│           │   │   │   ├── ethernet
│           │   │   │   │   └── intel
│           │   │   │   │       └── e1000
│           │   │   │   │           └── e1000.ko
│           │   │   │   └── fjes
│           │   │   │       └── fjes.ko
│           │   │   ├── scsi
│           │   │   │   ├── iscsi_boot_sysfs.ko
│           │   │   │   ├── sd_mod.ko
│           │   │   │   ├── sr_mod.ko
│           │   │   │   └── virtio_scsi.ko
│           │   │   └── virtio
│           │   │       ├── virtio.ko
│           │   │       ├── virtio_pci.ko
│           │   │       └── virtio_ring.ko
│           │   ├── fs
│           │   │   └── xfs
│           │   │       └── xfs.ko
│           │   ├── lib
│           │   │   ├── crc-t10dif.ko
│           │   │   └── libcrc32c.ko
│           │   └── net
│           │       ├── 802
│           │       │   └── stp.ko
│           │       ├── bridge
│           │       │   └── bridge.ko
│           │       └── llc
│           │           └── llc.ko
│           ├── modules.alias
│           ├── modules.alias.bin
│           ├── modules.builtin
│           ├── modules.builtin.bin
│           ├── modules.dep
│           ├── modules.dep.bin
│           ├── modules.devname
│           ├── modules.order
│           ├── modules.softdep
│           ├── modules.symbols
│           └── modules.symbols.bin
├── lib64
│   ├── ld-2.17.so
│   ├── ld-linux-x86-64.so.2 -> ld-2.17.so
│   ├── libc-2.17.so
│   ├── libc.so.6 -> libc-2.17.so
│   ├── libm-2.17.so
│   └── libm.so.6 -> libm-2.17.so
├── proc
├── root
├── run
├── sbin
│   ├── adjtimex -> ../bin/busybox
│   ├── arp -> ../bin/busybox
│   ├── bootchartd -> ../bin/busybox
│   ├── depmod -> ../bin/busybox
│   ├── fdisk -> ../bin/busybox
│   ├── freeramdisk -> ../bin/busybox
│   ├── fsck -> ../bin/busybox
│   ├── getty -> ../bin/busybox
│   ├── halt -> ../bin/busybox
│   ├── hwclock -> ../bin/busybox
│   ├── ifconfig -> ../bin/busybox
│   ├── ifdown -> ../bin/busybox
│   ├── ifenslave -> ../bin/busybox
│   ├── ifup -> ../bin/busybox
│   ├── init -> ../bin/busybox
│   ├── insmod -> ../bin/busybox
│   ├── ip -> ../bin/busybox
│   ├── ipaddr -> ../bin/busybox
│   ├── iplink -> ../bin/busybox
│   ├── ipneigh -> ../bin/busybox
│   ├── iproute -> ../bin/busybox
│   ├── iprule -> ../bin/busybox
│   ├── iptunnel -> ../bin/busybox
│   ├── klogd -> ../bin/busybox
│   ├── loadkmap -> ../bin/busybox
│   ├── logread -> ../bin/busybox
│   ├── lsmod -> ../bin/busybox
│   ├── makedevs -> ../bin/busybox
│   ├── mdev -> ../bin/busybox
│   ├── mkfs.ext2 -> ../bin/busybox
│   ├── modinfo -> ../bin/busybox
│   ├── modprobe -> ../bin/busybox
│   ├── pivot_root -> ../bin/busybox
│   ├── poweroff -> ../bin/busybox
│   ├── reboot -> ../bin/busybox
│   ├── rmmod -> ../bin/busybox
│   ├── route -> ../bin/busybox
│   ├── runlevel -> ../bin/busybox
│   ├── setconsole -> ../bin/busybox
│   ├── sulogin -> ../bin/busybox
│   ├── switch_root -> ../bin/busybox
│   ├── sysctl -> ../bin/busybox
│   ├── syslogd -> ../bin/busybox
│   ├── tunctl -> ../bin/busybox
│   ├── udhcpc -> ../bin/busybox
│   ├── vconfig -> ../bin/busybox
│   └── zcip -> ../bin/busybox
├── sys
├── sysroot
├── tmp
├── usr
│   ├── bin
│   │   ├── [ -> ../../bin/busybox
│   │   ├── [[ -> ../../bin/busybox
│   │   ├── awk -> ../../bin/busybox
│   │   ├── basename -> ../../bin/busybox
│   │   ├── bunzip2 -> ../../bin/busybox
│   │   ├── bzcat -> ../../bin/busybox
│   │   ├── bzip2 -> ../../bin/busybox
│   │   ├── cal -> ../../bin/busybox
│   │   ├── chrt -> ../../bin/busybox
│   │   ├── chvt -> ../../bin/busybox
│   │   ├── cksum -> ../../bin/busybox
│   │   ├── clear -> ../../bin/busybox
│   │   ├── cmp -> ../../bin/busybox
│   │   ├── comm -> ../../bin/busybox
│   │   ├── crontab -> ../../bin/busybox
│   │   ├── cryptpw -> ../../bin/busybox
│   │   ├── cut -> ../../bin/busybox
│   │   ├── dc -> ../../bin/busybox
│   │   ├── deallocvt -> ../../bin/busybox
│   │   ├── diff -> ../../bin/busybox
│   │   ├── dirname -> ../../bin/busybox
│   │   ├── dos2unix -> ../../bin/busybox
│   │   ├── du -> ../../bin/busybox
│   │   ├── env -> ../../bin/busybox
│   │   ├── envdir -> ../../bin/busybox
│   │   ├── envuidgid -> ../../bin/busybox
│   │   ├── expand -> ../../bin/busybox
│   │   ├── expr -> ../../bin/busybox
│   │   ├── find -> ../../bin/busybox
│   │   ├── fold -> ../../bin/busybox
│   │   ├── free -> ../../bin/busybox
│   │   ├── fuser -> ../../bin/busybox
│   │   ├── groups -> ../../bin/busybox
│   │   ├── head -> ../../bin/busybox
│   │   ├── hexdump -> ../../bin/busybox
│   │   ├── hostid -> ../../bin/busybox
│   │   ├── id -> ../../bin/busybox
│   │   ├── install -> ../../bin/busybox
│   │   ├── killall -> ../../bin/busybox
│   │   ├── last -> ../../bin/busybox
│   │   ├── less -> ../../bin/busybox
│   │   ├── logger -> ../../bin/busybox
│   │   ├── logname -> ../../bin/busybox
│   │   ├── lpq -> ../../bin/busybox
│   │   ├── lpr -> ../../bin/busybox
│   │   ├── lsof -> ../../bin/busybox
│   │   ├── lspci -> ../../bin/busybox
│   │   ├── md5sum -> ../../bin/busybox
│   │   ├── mesg -> ../../bin/busybox
│   │   ├── microcom -> ../../bin/busybox
│   │   ├── mkfifo -> ../../bin/busybox
│   │   ├── mkpasswd -> ../../bin/busybox
│   │   ├── nc -> ../../bin/busybox
│   │   ├── nmeter -> ../../bin/busybox
│   │   ├── nohup -> ../../bin/busybox
│   │   ├── nslookup -> ../../bin/busybox
│   │   ├── openvt -> ../../bin/busybox
│   │   ├── passwd -> ../../bin/busybox
│   │   ├── patch -> ../../bin/busybox
│   │   ├── pgrep -> ../../bin/busybox
│   │   ├── pkill -> ../../bin/busybox
│   │   ├── printf -> ../../bin/busybox
│   │   ├── pscan -> ../../bin/busybox
│   │   ├── readlink -> ../../bin/busybox
│   │   ├── realpath -> ../../bin/busybox
│   │   ├── renice -> ../../bin/busybox
│   │   ├── reset -> ../../bin/busybox
│   │   ├── resize -> ../../bin/busybox
│   │   ├── rpm2cpio -> ../../bin/busybox
│   │   ├── runsv -> ../../bin/busybox
│   │   ├── runsvdir -> ../../bin/busybox
│   │   ├── rx -> ../../bin/busybox
│   │   ├── script -> ../../bin/busybox
│   │   ├── seq -> ../../bin/busybox
│   │   ├── setkeycodes -> ../../bin/busybox
│   │   ├── setsid -> ../../bin/busybox
│   │   ├── setuidgid -> ../../bin/busybox
│   │   ├── sha1sum -> ../../bin/busybox
│   │   ├── sha256sum -> ../../bin/busybox
│   │   ├── sha3sum -> ../../bin/busybox
│   │   ├── sha512sum -> ../../bin/busybox
│   │   ├── softlimit -> ../../bin/busybox
│   │   ├── sort -> ../../bin/busybox
│   │   ├── split -> ../../bin/busybox
│   │   ├── strings -> ../../bin/busybox
│   │   ├── sum -> ../../bin/busybox
│   │   ├── tac -> ../../bin/busybox
│   │   ├── tail -> ../../bin/busybox
│   │   ├── tcpsvd -> ../../bin/busybox
│   │   ├── tee -> ../../bin/busybox
│   │   ├── telnet -> ../../bin/busybox
│   │   ├── test -> ../../bin/busybox
│   │   ├── tftp -> ../../bin/busybox
│   │   ├── time -> ../../bin/busybox
│   │   ├── top -> ../../bin/busybox
│   │   ├── tr -> ../../bin/busybox
│   │   ├── traceroute -> ../../bin/busybox
│   │   ├── traceroute6 -> ../../bin/busybox
│   │   ├── truncate -> ../../bin/busybox
│   │   ├── tty -> ../../bin/busybox
│   │   ├── ttysize -> ../../bin/busybox
│   │   ├── udpsvd -> ../../bin/busybox
│   │   ├── unexpand -> ../../bin/busybox
│   │   ├── uniq -> ../../bin/busybox
│   │   ├── unix2dos -> ../../bin/busybox
│   │   ├── unlink -> ../../bin/busybox
│   │   ├── unzip -> ../../bin/busybox
│   │   ├── uptime -> ../../bin/busybox
│   │   ├── users -> ../../bin/busybox
│   │   ├── uudecode -> ../../bin/busybox
│   │   ├── uuencode -> ../../bin/busybox
│   │   ├── vlock -> ../../bin/busybox
│   │   ├── wc -> ../../bin/busybox
│   │   ├── wget -> ../../bin/busybox
│   │   ├── who -> ../../bin/busybox
│   │   ├── whoami -> ../../bin/busybox
│   │   ├── whois -> ../../bin/busybox
│   │   ├── xargs -> ../../bin/busybox
│   │   └── yes -> ../../bin/busybox
│   ├── lib
│   ├── lib64
│   ├── sbin
│   │   ├── addgroup -> ../../bin/busybox
│   │   ├── add-shell -> ../../bin/busybox
│   │   ├── adduser -> ../../bin/busybox
│   │   ├── arping -> ../../bin/busybox
│   │   ├── brctl -> ../../bin/busybox
│   │   ├── chpasswd -> ../../bin/busybox
│   │   ├── chroot -> ../../bin/busybox
│   │   ├── crond -> ../../bin/busybox
│   │   ├── delgroup -> ../../bin/busybox
│   │   ├── deluser -> ../../bin/busybox
│   │   ├── dnsd -> ../../bin/busybox
│   │   ├── ether-wake -> ../../bin/busybox
│   │   ├── fakeidentd -> ../../bin/busybox
│   │   ├── httpd -> ../../bin/busybox
│   │   ├── inetd -> ../../bin/busybox
│   │   ├── killall5 -> ../../bin/busybox
│   │   ├── loadfont -> ../../bin/busybox
│   │   ├── lpd -> ../../bin/busybox
│   │   ├── ntpd -> ../../bin/busybox
│   │   ├── rdate -> ../../bin/busybox
│   │   ├── readahead -> ../../bin/busybox
│   │   ├── remove-shell -> ../../bin/busybox
│   │   ├── sendmail -> ../../bin/busybox
│   │   ├── setfont -> ../../bin/busybox
│   │   ├── setlogcons -> ../../bin/busybox
│   │   ├── svlogd -> ../../bin/busybox
│   │   ├── telnetd -> ../../bin/busybox
│   │   └── tftpd -> ../../bin/busybox
│   └── share
└── var

```

系统启动的init脚本，先做一些初始化工作再最后调用busybox的init
```text
[ming@centos7 initrd-busybox]$ cat ./init 
#!/bin/sh

echo "Heath customised init..."
echo "Mount the /proc and /sys filesystems"
mount -t proc proc /proc
mount -t sysfs sys /sys

echo "Creating /dev"
mount -o mode=0755 -t tmpfs /dev /dev
mkdir /dev/pts
mount -t devpts -o gid=5,mode=620 /dev/pts /dev/pts
mkdir /dev/shm
mkdir /dev/mapper
echo "Creating initial device nodes"
mknod /dev/null c 1 3
mknod /dev/zero c 1 5
mknod /dev/systty c 4 0
mknod /dev/tty c 5 0
mknod /dev/console c 5 1
mknod /dev/ptmx c 5 2
mknod /dev/rtc c 10 135
mknod /dev/tty0 c 4 0
mknod /dev/tty1 c 4 1
mknod /dev/tty2 c 4 2
mknod /dev/tty3 c 4 3
mknod /dev/tty4 c 4 4
mknod /dev/tty5 c 4 5
mknod /dev/tty6 c 4 6
mknod /dev/ttyS0 c 4 64
mknod /dev/ttyS1 c 4 65


mount -t usbfs /proc/bus/usb /proc/bus/usb

modprobe serio_raw
modprobe pata_acpi
modprobe libahci
modprobe ahci
modprobe sd_mod
modprobe libata
modprobe ata_piix
modprobe xfs
modprobe dm_mod
modprobe e1000



echo "Mounting root filesystem"


echo "switch_root and run /sbin/init"
#exec switch_root /mnt/root /sbin/init
exec /sbin/init

```

inittab文件
```text
[ming@centos7 initrd-busybox]$ cat etc/inittab 
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/login
#::askfirst:/bin/sh
#::respawn:/sbin/getty 9600 tty1
#::respawn:/sbin/getty 9600 tty2
#::respawn:/sbin/getty 9600 tty3
::shutdown:/bin/umount -a -r
::ctrlaltdel:/sbin/reboot

```


```text
[ming@centos7 initrd-busybox]$ cat etc/init.d/rcS 
#!/bin/sh

echo "Run /etc/init.d/rcS"

if [ ! -e /proc/mounts ]; then
	mount -n -t proc proc /proc
	mount -n -t sysfs sysfs /sys
fi

echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s

mount -a

mkdir -p /var/log
mkdir -p /var/run
mkdir -p /var/lock
mkdir -p /var/local
mkdir -p /var/cache
mkdir -p /var/local
mkdir -p /var/opt
mkdir -p /var/tmp
mkdir -p /var/spool
touch /var/log/lastlog
mkdir -p /tmp/root

/sbin/klogd -c 1
/sbin/syslogd

echo "Initializing ..."
mount -o remount,rw /

#ifconfig  lo  127.0.0.1/24

echo "Done."
exit

```

mdev的配置文件
```text
[ming@centos7 initrd-busybox]$ cat etc/mdev.conf 
# https://git.busybox.net/busybox/tree/docs/mdev.txt?h=1_26_stable

# for debugging
# -.*    root:root	0666 *{ date; set; echo; } >>/tmp/log

# null may already exist; therefore ownership has to be changed with command
null                    root:root       666 @/bin/chmod 666 "$MDEV"
zero                    root:root       666
full                    root:root       666
random                  root:root       666
urandom                 root:root       666
hwrandom                root:root       666

kmsg                    root:root       600
mem                     root:kmem       640

# console may already exist; therefore ownership has to be changed with command
console                 root:tty        600 @/bin/chmod 600 "$MDEV"
ptmx                    root:tty        666
pty.*                   root:tty        666
tty[a-z][0-9a-f]	root:tty        666

microcode               root:root       600 =cpu/

# Typical devices
tty                     root:tty        666
tty[0-9]		root:tty        620
vcsa?[0-9]*             root:tty        620
ttyS[0-9]+              root:dialout    660

# block devices
ram[0-9]+		root:disk       660
loop[0-9]+		root:disk	660
sd[a-z].*               root:disk       660
hd[a-z].*		root:disk       660

# net and misc
tun[0-9]*		root:root	660 =net/
tap[0-9]*		root:root	660 =net/
fuse                    root:root	660
device-mapper		root:root	600 >mapper/control

# alsa sound devices and audio stuff
pcm.*		root:audio 660 =snd/
control.*	root:audio 660 =snd/
midi.*		root:audio 660 =snd/
seq		root:audio 660 =snd/
timer		root:audio 660 =snd/

# input stuff
event[0-9]+	root:root 660 =input/
mice		root:root 660 =input/
mouse[0-9]+	root:root 660 =input/


```


3. 制作完initramfs，配置grub2增加我们的内核和initramfs就可以了。
修改/etc/grub.d/40_custom 增加自定义的entry，然后  grub2-mkconfig -o /boot/grub2/grub.cfg 更新
```text
menuentry 'CentOS Linux (busybox) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-busybox-advanced-0940d618-e2e8-4621-9688-24e3cbe4c13c' {
	load_video
	insmod gzio
	insmod part_msdos
	insmod xfs
	set root='hd0,msdos1'
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint-bios=hd0,msdos1 --hint-efi=hd0,msdos1 --hint-baremetal=ahci0,msdos1 --hint='hd0,msdos1'  204d6bba-f874-4720-a784-24e99d6d28a6
	else
	  search --no-floppy --fs-uuid --set=root 204d6bba-f874-4720-a784-24e99d6d28a6
	fi
	linux16 /vmlinuz-3.10.0-514.10.2.el7.x86_64 console=ttyS0 root=/dev/root ro crashkernel=auto rd.lvm.lv=cl_centos7/root rd.lvm.lv=cl_centos7/swap rhgb
  initrd16 /initramfs-busybox.img
}

```

z
