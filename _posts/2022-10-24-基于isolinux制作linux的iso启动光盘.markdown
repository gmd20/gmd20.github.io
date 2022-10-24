参考 https://wiki.syslinux.org/wiki/index.php?title=ISOLINUX


```text
dnf install syslinux  syslinux-nonlinux
dns install xorriso

cp /boot/vmlinuz-4.18.0-372.26.1.el8_6.x86_64  CD_root/isolinux/vmlinuz
cp initramfs.img CD_root/isolinux/initramfs.img
cp /usr/share/syslinux/isolinux.bin ./CD_root/isolinux
cp /usr/share/syslinux/ldlinux.c32 ./CD_root/isolinux


# find CD_root/
CD_root/
CD_root/isolinux
CD_root/isolinux/isolinux.bin
CD_root/isolinux/ldlinux.c32
CD_root/isolinux/initramfs.img
CD_root/isolinux/vmlinuz
CD_root/isolinux/isolinux.cfg


# cat ./CD_root/isolinux/isolinux.cfg
prompt 1
timeout 1
default test

label test
	MENU LABEL test
	kernel vmlinuz
	append initrd=initramfs.img ro rhgb quiet vga=0x318 logo.nologo console=tty0 console=ttyS0,115200n8 LANG=en_US.UTF-8


mkisofs -o test.iso \
   -b isolinux/isolinux.bin -c isolinux/boot.cat \
   -no-emul-boot -boot-load-size 4 -boot-info-table \
   CD_root

```
