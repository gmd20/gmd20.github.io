1. 磁盘管理器里面压缩某个盘，留出空间创建，Ubunut的分区
--------------------------------------------------------
我的笔记本一个128GB的SSD已经有EFI分区，安装了windows10了。机械硬盘是1TB的，
我就打算在机械硬盘上面也创建一个EFI分区，安装完这个笔记本启动时按F12进入UEFI
启动菜单，可以选择从那个磁盘的UEFI启动。ubuntu和windows安装在不同的磁盘上面，
都可以独立运行，一个硬盘坏了另外一个也不会受影响。如果都安装到一个磁盘EFI分区，
怕出问题了麻烦。
在机械硬盘最后一个盘空间之前配置的比较大，可以压缩出100GB来安装Ubuntu，操作很简单
在磁盘管理器里面，右键最后一个盘，选 “压缩卷”，减少100GB就可以了。这100GB后面
会分成一个250MB的EFI系统分区（windows的文档说有的磁盘扇区最小为4K，所以最小的FAT32
分区的磁盘只能到256MB，所以推荐把这个分区设置为260MB。但有的磁盘可以支持512个字节
的扇区的话，家里的机械硬盘windows的安装程序默认建立的EFI分区只有100MB大小。不过
这个联想笔记本SSD上面的EFI分区设置的就是260MB。），剩下的都作为ubuntu的ext4分区。

这个操作用diskpart命令应该也是可以得，不过我更喜欢GUI一些。这个无损压缩文件系统
的空间，linux的也支持，不过操作没有windows这么方便。


2. 使用windows自带的分区命令diskpart创建一个250MB的efi系统分区
--------------------------------------------------------------
参考 https://technet.microsoft.com/en-us/library/Hh825686.aspx

这个EFI分区没法在磁盘管理器里面创建，只能通过管理员权限的命令终端的diskpart命令
来进行了。创建的时候格式化成FAT32的格式，这个是UEFI要求的EFI分区格式吧。

这个在安装windows 10系统的时候，它也不让你自己创建EFI分区的，只能自己指定大小。
不过如果你选择一个没有分区的磁盘，在UEFI模式下它的安装程序默认会帮你建一个EFI系统
分区和WINRE分区，还有MSR保留分区之类的。不过它默认的分区不一定符合自己的胃口吧。
还是安装系统之前，自己用diskpart命令来划分一下比较好。
Ubuntu的安装程序里面GUI上面就可以自己创建EFI类型分区，自由设置大小各种格式等。
不知道Windows怎么这么死板，一般人肯定不懂怎么建立这个EFI分区和调整大小了。

这个大小可能设置为260MB更好吧，应该在所有的磁盘都支持。不过如果磁盘支持，配置
成100MB也是可以得，参考前面的说明。

```text
PS C:\WINDOWS\system32> diskpart

Microsoft DiskPart 版本 10.0.15063.0

DISKPART> list disk

  磁盘 ###  状态           大小     可用     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  磁盘 0    联机              931 GB   100 GB        *
  磁盘 1    联机              119 GB      0 B        *

DISKPART> select disk 0

磁盘 0 现在是所选磁盘。

DISKPART> list partition

  分区 ###       类型              大小     偏移量
  -------------  ----------------  -------  -------
  分区      1    保留                 128 MB    17 KB
  分区      2    主要                 200 GB   129 MB
  分区      3    主要                 200 GB   200 GB
  分区      4    主要                 431 GB   400 GB

DISKPART> create partition efi size=250

DiskPart 成功地创建了指定分区。

DISKPART> exit

退出 DiskPart...


**前面忘了格式化了，又要重做一遍**

PS C:\WINDOWS\system32> diskpart

Microsoft DiskPart 版本 10.0.15063.0

DISKPART> list disk

  磁盘 ###  状态           大小     可用     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  磁盘 0    联机              931 GB    99 GB        *
  磁盘 1    联机              119 GB      0 B        *

DISKPART> select disk 0

磁盘 0 现在是所选磁盘。

DISKPART> list parttition

Microsoft DiskPart 版本 10.0.15063.0

DISK        - 显示磁盘列表。例如，LIST DISK。
PARTITION   - 显示所选磁盘上的分区列表。
              例如，LIST PARTITION。
VOLUME      - 显示卷列表。例如，LIST VOLUME。
VDISK       - 显示虚拟磁盘列表。

DISKPART> list partition

  分区 ###       类型              大小     偏移量
  -------------  ----------------  -------  -------
  分区      1    保留                 128 MB    17 KB
  分区      2    主要                 200 GB   129 MB
  分区      3    主要                 200 GB   200 GB
  分区      4    主要                 431 GB   400 GB
  分区      5    系统                 250 MB   831 GB

DISKPART>

DISKPART>

DISKPART> select partition 5

分区 5 现在是所选分区。

DISKPART> format quick fs=fat32 label="System"

  100 百分比已完成

DiskPart 成功格式化该卷。

```


3. 在windows里面访问EFI分区
----------------------------
```text
PS C:\WINDOWS\system32> mountvol  /?
创建、删除或列出卷装入点。

MOUNTVOL [drive:]path VolumeName
MOUNTVOL [drive:]path /D
MOUNTVOL [drive:]path /L
MOUNTVOL [drive:]path /P
MOUNTVOL /R
MOUNTVOL /N
MOUNTVOL /E
MOUNTVOL drive: /S

    path        指定装入点将驻留的现有 NTFS 目录。
    VolumeName  指定装入点的目标的卷名称。
    /D          从指定的目录中删除卷装入点。
    /L          列出指定目录的已装入的卷名称。
    /P          从指定目录删除卷装入点，卸下此卷并使此卷无法装入。你可以创建
                一个卷来再次使此卷可以装入。
    /R          删除不在系统中的、卷的装入点目录和注册表设置。
    /N          禁用新卷的自动装入。
    /E          再次启用新卷的自动装入。
    /S          将 EFI 系统分区装载到提供的驱动器。

```

有两个磁盘每个都有EFI分区，这个mountvol命令好像没法指定mount那个efi分区。
只能用diskpart命令的assign子命令来操作了。
```text
PS C:\WINDOWS\system32> diskpart

Microsoft DiskPart 版本 10.0.15063.0

DISKPART> list disk

  磁盘 ###  状态           大小     可用     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  磁盘 0    联机              931 GB    99 GB        *
  磁盘 1    联机              119 GB      0 B        *

DISKPART> select disk 0

磁盘 0 现在是所选磁盘。

  分区 ###       类型              大小     偏移量
  -------------  ----------------  -------  -------
  分区      1    保留                 128 MB    17 KB
  分区      2    主要                 200 GB   129 MB
  分区      3    主要                 200 GB   200 GB
  分区      4    主要                 431 GB   400 GB
  分区      5    系统                 250 MB   831 GB

DISKPART> select partition 5

分区 5 现在是所选分区。

DISKPART> assign letter=H

DiskPart 成功地分配了驱动器号或装载点。
PS C:\WINDOWS\system32> diskpart

Microsoft DiskPart 版本 10.0.15063.0

DISKPART> list disk

  磁盘 ###  状态           大小     可用     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  磁盘 0    联机              931 GB    99 GB        *
  磁盘 1    联机              119 GB      0 B        *

DISKPART> select disk 0

磁盘 0 现在是所选磁盘。

DISKPART> list partition

  分区 ###       类型              大小     偏移量
  -------------  ----------------  -------  -------
  分区      1    保留                 128 MB    17 KB
  分区      2    主要                 200 GB   129 MB
  分区      3    主要                 200 GB   200 GB
  分区      4    主要                 431 GB   400 GB
  分区      5    系统                 250 MB   831 GB

DISKPART> select partition 5

分区 5 现在是所选分区。

DISKPART> assign letter=H

DiskPart 成功地分配了驱动器号或装载点。

DISKPART> exit

退出 DiskPart...
```

在资源管理器里面已经看到 EFI的分区对应的盘了。 不过要想访问里面的内容，需要有
管理员权限才行, 可以在以管理员身份运行的命令提示符窗口里面强制结束explorer.exe
再以管理员身份执行exploer.exe。
```bat
taskkill /im explorer.exe /f
explorer.exe
```

也可以在拥有管理员权限命令提示符窗口里面用 xcopy命令来复制文件。
或者在开始菜单里面搜索某个应用比如记事本 notepad，右键选择以管理员身份启动，
然后选择菜单 文件->打开，在打开的文件对话框里面复制粘贴文件。


4. 配置Ubuntu的ISO镜像的UEFI启动项，GRUB2的配置
---------------------------------------------s-
https://help.ubuntu.com/community/Grub2/ISOBoot
https://wiki.archlinux.org/index.php/GRUB_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)
https://help.ubuntu.com/community/Grub2/Troubleshooting
https://www.gnu.org/software/grub/manual/grub.html

我把Ubuntu的iso安装镜像文件放在windows系统的一个ntfs格式的盘里面。
位置是： D:\ubuntu-17.04-desktop-amd64.iso
对应的diskpart的分区位置是 disk 1 partion 4上面吧。
可以在grub终端使用命令加载iso文件，启动ubuntu安装live CD的吧，

常用的grub命令：
insmod  加载需要的模块，比如insmod ntfs以支持ntfs文件系统，不过好像默认已经支持了
ls    列出磁盘位置和文件，用法 ls (hd0,gpt2), ls (hd0,gpt3)/ubuntu  等
configfile   加载某个配置菜单文件。用法 configfile (hd1,gpt4)/grub.cfg
cat  查看文件内容

我事先创建一个grub.cfg文件配置在D盘的根目录，看看到时能不能手工加载，这样可以少
手工输入一些命令。

**注意，这个iso文件不要放在和ubuntu的根分区的同一个磁盘上面，不然安装的时候
ubuntu要格式化ext4文件系统或者新建分区，就会提示ISO文件挂载点占用了/dev/sda设备，
没法卸载设备，ubuntu需要先卸载磁盘，才能更新分区表和格式化。**

```text
# Timeout for menu
set timeout=5

# Set default boot entry as Entry 0
set default=0

# (0) Ubuntu
menuentry "Ubuntu 17.04.2 ISO" {
    insmod part_gpt
    insmod ntfs
    insmod iso9660
    # search -f -set=root /ubuntu-17.04-desktop-amd64.iso
    set isofile="/Ubuntu/ubuntu-17.04-desktop-amd64.iso"
    loopback loop (hd1,gpt4)$isofile
    linux (loop)/casper/vmlinuz.efi boot=casper iso-scan/filename=$isofile noprompt noeject
    initrd (loop)/casper/initrd.lz
}

## (1) Windows
menuentry "Windows 10" {
    insmod part_gpt
    insmod fat
    insmod search_fs_uuid
    insmod chain

    # search --fs-uuid --set=root $hints_string $fs_uuid
    #  $hints_string and $fs_uuid is from these commands:
    #    grub-probe --target=fs_uuid $esp/EFI/Microsoft/Boot/bootmgfw.efi
    #    grub-probe --target=hints_string $esp/EFI/Microsoft/Boot/bootmgfw.efi
    #  $esp is the EFI system partttion mount path
    # example:
    # search --fs-uuid --no-floppy --set=root --hint-bios=hd0,gpt2 --hint-efi=hd0,gpt2 --hint-baremetal=ahci0,gpt2 763A-9CB6

    # search --set=root --file /EFI/Microsoft/Boot/bootmgfw.efi

    set root='(hd1,gpt3)'
    chainloader /EFI/Microsoft/Boot/bootmgfw.efi
}

### (3) Restart
menuentry "System restart" {
	echo "System rebooting..."
	reboot
}
```


5. 解压iso，把里面grub启动EFI程序复制到前面刚刚新建的EFI分区里面。
-----------------------------------------------------------------
把ISO光盘里面的boot目录和EFI目录复制到EFI分区（H盘）里面去

在git bash列出来的EFI分区的内容如下。
```text
$ find .
.
./boot
./boot/grub
./boot/grub/efi.img
./boot/grub/font.pf2
./boot/grub/grub.cfg
./boot/grub/loopback.cfg
./boot/grub/x86_64-efi
./boot/grub/x86_64-efi/acpi.mod
...
./boot/grub/x86_64-efi/zfscrypt.mod
./EFI
./EFI/BOOT
./EFI/BOOT/BOOTx64.EFI
./EFI/BOOT/grubx64.efi
```


6. 启动时按F12进入uefi启动菜单，选择从机械硬盘启动，可以进入grub了
------------------------------------------------------------------
注意要先在bios的 “security”里面禁用“secure boot” 才能加载各个grub模块，不然ls这些命令都没法
使用。这个估计主板bios校验efi程序，不然没经过验证的程序启动。按照网上的说法好像
说grub有提供另外一个efi镜像给绕过这个secure boot机制的检查，但我在ubuntu里面
没看到。所以还是现在bios禁用一下，再开始使用grub引导。

显示的grub菜单，应该是 EFI分区的./boot/grub/grub.cfg，
里面有几个Try Ubuntu，Install Ubuntu的LiveCD的启动项，但我这个EFI分区比较小，
是没有把ubuntu的vmlinuz.efi initrd.lz等文件复制到这个分区来的，肯定是启动不了。
我是希望他加载ntfs文件系统磁盘上面的iso来启动的。

按下c键，进入grub终端， 用ls 命令查看磁盘名字，可以看到是(hd0,gpt2) 这样的。
ismod ntfs    好像默认已经加载了
ls (hd1,gpt4)/  可以看到之前创建的grub.cfg 文件了。
configfile  (hd1,gpt4)/grub.cfg   加载配置文件，就重新弹出我们配置菜单了。
不过之前磁盘写法不对，所以还要修改成(hd0,gpt2)这种
这个grub的配置文件里面应该是可以指定磁盘分区的明确的UUID的，windows通过diskpart
查看这个UUID的吧，linux上面可以通过blkid命令。


7. 成功进入Ubuntu的initrd，但之后老报告iso-scan/filename那里制定的iso文件找不到
-------------------------------------------------------------------
可以知道grub的那个loopback命令肯定已经正确mount了iso文件了。是Ubuntu的安装程序
提示的错误。这个有点奇怪，看上去Ubuntu的安装程序没有再ntfs文件系统下面搜索这个iso。
解开iso镜像里面安装脚本看看, 是initrd\scripts\casper-premount\20iso_scan 这个脚本
输出的错误，就是find_path失败了。查看了一下脚本，应该是支持mount ntfs类型的磁盘
分区的，奇怪了。再次启动，仔细检查打印出来日志，发现都没有到mount那一步
在initrd命令提示符下面检查/sys/block下面的块设备都没有/dev/sda这些磁盘出现，
再看屏幕的信息，有ATA queue tieout的错误信息。应该是磁盘驱动的错误了，
SATA相关的吧，到bios 里面查看一下SATA模式，现在设置为“Intel - RST -Prenium”，
想起网上有人说到关闭“英特尔® 快速存储技术（英特尔® RST）” 模式的，看来Ubuntu不
支持这个技术，只能把SATA模式改成AHCI。 保存之后推出重启安装，这次终于可以进入
ubuntu的图形界面了。这个Linux这么多年了还是有这种兼容问题啊，我记得以前的旧电脑
也是对SATA模式的支持有问题。还是没法跟上时代啊。为什么不默认把intel的这个驱动加
上来呢。改成这个sata AHCI模式之后，装好的windows 10又进不去，windows安装好之后
也不能同时兼容两种模式，看来以后要用linux每次都要进入bios里面改这个sata模式了
，用完了又要改回来，超级麻烦。

安装的时候，发现ubuntu没法更新这个磁盘的分区表，因为同一个磁盘的iso文件还挂载着，没法
卸载设备。看来这个iso镜像还是放到和这个ubuntu安装分区不同的磁盘上面比较好。试了
先退出来 在windows里面创建一好NTFS分区，在进入ubuntu那里再格式化成ext4这样也还是不行。
好在我这个笔记本有两块磁盘，先把iso安装镜像放到另外一个SSD磁盘就好了。再相应的
改一下前面的grub.cfg配置文件。如果电脑没有两块硬盘，估计是硬盘安装不了，只能
用U盘安装了。我看网上别人都是建立一个很大EFI分区，然后把iso光盘全部解压出来
放到里面。我是不想设置一个这么大EFI分区才使用iso文件直接安装的方法。如果把iso光盘
解压到同一个磁盘的ntfs分区，不知道安装时能不能创建更新分区表？有时间的可以试一下。

ubuntu安装的时候，不要把选择把grub启动引导安装到windows 10的那个window boot manager一起，
选择安装到第二块磁盘的EFI分区上面。

**不过ubuntu还是把grub给安装到windows 10的那个EFI分区里面去了**


8. 整理一下UEFI启动项和各个EFI分区
-----------------------------------
首先把ubuntu的grub EFI启动移到第二个磁盘的EFI系统分区上面来。

windows 10的那个EFI系统分区是 /dev/nvme0n1p1，刚刚新建的是 /dev/sda5。
linux上面操作EFI分区还是比较方便的，直接把对应的分区mount了，
然后用grub-install安装 grub 的efi启动项。
```text
sudo mount /dev/sda5 /mnt
sudo grub-install --target=x86_64-efi --efi-directory=/mnt
```

做完可以检查一下EFI目录下面有没有ubuntu在里面了。然后blkid查找一下分区的UUID，
修改一下/etc/fstab 文件里面的/boot/efi对应的挂载点，设置到新的/dev/sda5对应的
EFI分区，注释掉以前的，复制一行，修改一下UUID指向新的EFI分区的UUDI就可以了。
```text
:~$ sudo blkid /dev/sda5
/dev/sda5: LABEL="SYSTEM" UUID="5A19-A693" TYPE="vfat" PARTLABEL="EFI system partition" PARTUUID="1a0d511a-f6e6-4ad2-81f4-aa1579ad9212"
:~$ sudo blkid /dev/nvme0n1p1
/dev/nvme0n1p1: LABEL="SYSTEM_DRV" UUID="9A07-5FDE" TYPE="vfat" PARTLABEL="EFI system partition" PARTUUID="1d8469a1-177d-4c60-a36c-d5b3759d77dc"
```

参考 https://superuser.com/questions/930725/how-to-delete-os-from-boot-menu
windows平台等可以用easyUEFI工具，ubuntu平台使用efibootmgr命令，有的主板也可以直
接进入EFI version 2 shell 使用bcfg来配置。联想的这个笔记本只列出所有的EFI启动项，
然后可以调整一下顺序。这个启动菜单是保存在 UEFI的NVRAM里面的，主板的UEFI支持程序
自动扫描出来保存起来的吧。 efibootmgr其实是通过linux内核提供的/sys/fireware/efi/vars
文件操作接口来设置的，好像是哪个efi vars相关的驱动提供的支持。easyUEFI这些工具
很容易使用吧。不过没用过，看样子只要把*.efi文件放到EFI目录下，主板的UEFI就自动
扫描出来了。所以我也只是把EFI系统分区mount到linux下面，然后自己直接删掉里面的内容，
把windows使用的ESP分区EFI目录下面的ubuntu目录删掉，第二块磁盘EFI系统分区只
保留ubuntu目录，把之前自己复制的光盘里面的grub的BOOT目录那些都删掉。最后的EFI分区
内容是这样的。

```text
$ tree /boot/efi
/boot/efi/
└── EFI
    └── ubuntu
        ├── grub.cfg
        ├── grubx64.efi
        ├── mmx64.efi
        └── shimx64.efi
```

修改完重启可以正常进入ubuntu系统，按照网上说法有了shimx64.efi这个，也是可以在
bios里面 “secure mode”开启情况下也可以工作的。试了一下bios打开secure mode确实
可以正常进入ubuntu，不过安装第3方驱动的时候它会提示你把这个UEDI的安全模式给关
闭的，说是有可能第3方驱动可能不支持。不过SATA的AHCI模式还是要保留的，安装完
ubuntu还是不支持“ Intel® Rapid Start Technology”模式，按理来说这个Linux内核
应该是支持的了，估计是ubuntu的人没把驱动整合好，懒得去搞了。
https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/rst-linux-paper.pdf

可以看一下ubuntu的grub.cfg 配置文件，它在EFI
分区里面那个设置文件会加载/boot/grub下面的配置文件，然后都自己设置了windows 10
的启动项在里面了，可以参考学习一下。

```text
search.fs_uuid 0aa5cade-9f43-402d-af8a-3bc0b490c39c root hd0,gpt6
set prefix=($root)'/boot/grub'
configfile $prefix/grub.cfg
```

它的设置的windows 10的grub启动项
```text
menuentry 'Windows Boot Manager (on /dev/nvme0n1p1)' --class windows --class os $menuentry_id_option 'osprober-efi-9A07-5FDE' {
	insmod part_gpt
	insmod fat
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root  9A07-5FDE
	else
	  search --no-floppy --fs-uuid --set=root 9A07-5FDE
	fi
	chainloader /EFI/Microsoft/Boot/bootmgfw.efi
}

menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-0aa5cade-9f43-402d-af8a-3bc0b490c39c' {
	recordfail
	load_video
	gfxmode $linux_gfx_mode
	insmod gzio
	if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
	insmod part_gpt
	insmod ext2
	set root='hd0,gpt6'
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint-bios=hd0,gpt6 --hint-efi=hd0,gpt6 --hint-baremetal=ahci0,gpt6  0aa5cade-9f43-402d-af8a-3bc0b490c39c
	else
	  search --no-floppy --fs-uuid --set=root 0aa5cade-9f43-402d-af8a-3bc0b490c39c
	fi
	linux	/boot/vmlinuz-4.10.0-19-generic.efi.signed root=UUID=0aa5cade-9f43-402d-af8a-3bc0b490c39c ro  quiet splash $vt_handoff
	initrd	/boot/initrd.img-4.10.0-19-generic
}
```



9. 无线驱动，windows和ubuntu双系统的时间文件
--------------------------------------------

安装完Intel的无线网卡也工作不正常，在软件更新里面更新了驱动也不行，可能需要自己
手工安装一下，后面有需要再弄了。

还发现一个问题时间同步的问题，切换到ubuntu系统再切回来，肯定一个系统里面看到的
时间不对了，网上说是因为windows 往bios写如的是localtime本地直接，但linux等系统
现在默认写到bios里面的是utc time。需要有一方调整一下，主流标准是保存UTC时间了。
虽然linux修改比较简单，但都是windows 10版本了应该没有bug了吧，我还是改注册
表让windows保存成UTC时间到bios了, 把下面的内容放到*.reg文件导入或者直接在注册
表里面新建这个值都是可以的吧。
```text
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\TimeZoneInformation]
 "RealTimeIsUniversal"=dword:00000001
 ```

详细参考 https://wiki.archlinux.org/index.php/time

