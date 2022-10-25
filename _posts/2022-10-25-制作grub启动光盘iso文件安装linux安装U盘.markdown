# grub2需要额外安装的包
```text
dnf provides /usr/lib/grub/x86_64-efi
dnf install grub2-pc grub2-efi-x64 grub2-efi-x64-modules xorriso
mkdir iso
```

```text
常规的MBR磁盘分区，要预留2MB空间给grub吧，比如
# parted -a optimal -s /dev/sdc -- mklabel msdos \
    mkpart primary ext4 4096s 400MB \
    mkpart primary ext3 400MB -1s \
    set 1 boot on \
    print

复制grub到到硬盘MBR引导扇区, 常规的磁盘是这么安装grub，但grub2-mkrescue 会自动复制这个文件吧
# grub2-install --target=i386-pc --boot-directory=iso/boot /dev/sdc

复制grub的efi镜像到gpt磁盘esp分区。
# grub2-install --target=x86_64-efi --removable --no-nvram --boot-directory=iso/boot --efi-directory=iso
     grub2-install 现在好像不支持安装"x86_64-efi"了，可能是修改efi启动变量可能有什么问题之类，好像建议用efibootmgr
     --no-nvram 和 --removable 这两个参数也和efi变量有关系吧， --removable保证这个设备拿到别的机器依然可以启动？
等价于下面的手工复制efi程序
# mkdir -p iso/boot/efi/EFI/
# cp /boot/efi/EFI/rocky/grubx64.efi iso/boot/efi/EFI/

还有一个grub-mkimagec 创建grubx64.efi文件，有很多参数
# grub-mkimage -O x86_64-efi -o /tmp/grubx64.efi
```

# iso目录的是这样的
```text
# find iso
iso
iso/boot
iso/boot/vmlinuz
iso/boot/initramfs.img
iso/boot/grub
iso/boot/grub/grub.cfg
```

# grub配置文件的内容 iso/boot/grub/grub.cfg
```
function load_video {
  if [ x$feature_all_video_module = xy ]; then
    insmod all_video
  else
    insmod efi_gop
    insmod efi_uga
    insmod ieee1275_fb
    insmod vbe
    insmod vga
    insmod video_bochs
    insmod video_cirrus
  fi
}

serial --unit=0 --speed=115200 --word=8 --parity=no --stop=1
terminal_input serial console
terminal_output serial console
set timeout=1

default=0
menuentry 'linux' --class gnu-linux --class gnu --class os --unrestricted {
	load_video
	set gfxpayload=keep
	insmod gzio
	insmod part_msdos
	insmod ext2
	linux /boot/vmlinuz root=/dev/sda1 ro rhgb quiet vga=0x318 logo.nologo console=tty0 console=ttyS0,115200n8 LANG=en_US.UTF-8
	initrd /boot/initramfs.img
}
```

# 上面的grub2-install都不需要，grub2-mkrescue 会自动打包MBR和efi引导文件
```text
grub2-mkrescue -o grub.iso iso
```
最后iso文件结构是遮掩的
```text
./boot
./boot/grub
./boot/grub/fonts
./boot/grub/fonts/unicode.pf2
./boot/grub/grub.cfg
./boot/grub/i386-pc
./boot/grub/i386-pc/acpi.mod
./boot/grub/roms
./boot/grub/x86_64-efi
./boot/grub/x86_64-efi/acpi.mod
./boot/initramfs.img
./boot/vmlinuz
./boot.catalog
./efi.img
./mach_kernel
./System
./System/Library
./System/Library/CoreServices
./System/Library/CoreServices/.disk_label
./System/Library/CoreServices/.disk_label.contentDetails
./System/Library/CoreServices/boot.efi
./System/Library/CoreServices/SystemVersion.plist
./[BOOT]
./[BOOT]/1-Boot-NoEmul.img
./[BOOT]/2-Boot-NoEmul.img
```


# rufus把iso复制到U盘时要选择“dd方式保留原始文件系统”
用“iso到fat32格式u盘”最后grub引导会找到文件系统格式。好像rocky linux默认grub
好不支持fat32格式的文件系统，导致加载不了内核
制作出来的u盘，uefi和传统MBR引导都是可以的。
