windows 10工作很正常，但linux下面有问题， 估计是linux对这这种芯片的支持有问题？(这个好像跟实际的msata卡关，现在发现有一款在linux下不支持这个UAS，另另外一个正常)
内核日志看UAS工作不正常，看到网上很多人遇到类似的问题，只好把UAS特性给关闭了，当作普通的USB storage来用，   
但测试发现金庸UAS后如果插入的是USB3.0的接口还是工作的不正常的，还好插入USB2.0里面可以工作了，虽然读写速度可能有些下降，   
但还能正常使用算不错了。

dmesg 
=====
If UAS is not disabled, the scsi host adapter was reset again and again.
```text
[ 9328.819527] usb 2-1: new SuperSpeed USB device number 4 using xhci_hcd
[ 9328.839697] usb 2-1: New USB device found, idVendor=174c, idProduct=55aa
[ 9328.839704] usb 2-1: New USB device strings: Mfr=2, Product=3, SerialNumber=1
[ 9328.839708] usb 2-1: Product: KS-C7MG
[ 9328.839712] usb 2-1: Manufacturer: KINGSHARE
[ 9328.839716] usb 2-1: SerialNumber: 123456xxxxxx
[ 9328.842588] scsi host3: uas
[ 9328.845552] scsi 3:0:0:0: Direct-Access     KINGSHAR KS-C7MG          0    PQ: 0 ANSI: 6
[ 9328.846310] sd 3:0:0:0: Attached scsi generic sg1 type 0
[ 9328.847722] sd 3:0:0:0: [sdb] 7865424 512-byte logical blocks: (4.03 GB/3.75 GiB)
[ 9328.847835] sd 3:0:0:0: [sdb] Write Protect is off
[ 9328.847839] sd 3:0:0:0: [sdb] Mode Sense: 43 00 00 00
[ 9328.847997] sd 3:0:0:0: [sdb] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
[ 9328.849831]  sdb: sdb1
[ 9328.851238] sd 3:0:0:0: [sdb] Attached SCSI disk
[ 9360.591140] sd 3:0:0:0: [sdb] tag#3 uas_eh_abort_handler 0 uas-tag 4 inflight: CMD IN 
[ 9360.591148] sd 3:0:0:0: [sdb] tag#3 CDB: Read(10) 28 00 00 00 00 80 00 01 80 00
[ 9360.611132] scsi host3: uas_eh_device_reset_handler start
[ 9360.739466] usb 2-1: reset SuperSpeed USB device number 4 using xhci_hcd
[ 9360.761497] scsi host3: uas_eh_device_reset_handler success
[ 9391.319141] scsi host3: uas_eh_device_reset_handler start
[ 9391.319250] sd 3:0:0:0: [sdb] tag#0 uas_zap_pending 0 uas-tag 1 inflight: CMD 
[ 9391.319257] sd 3:0:0:0: [sdb] tag#0 CDB: Read(10) 28 00 00 00 00 80 00 01 80 00
[ 9391.447523] usb 2-1: reset SuperSpeed USB device number 4 using xhci_hcd
[ 9391.469599] scsi host3: uas_eh_device_reset_handler success
[ 9424.083624] scsi host3: uas_eh_device_reset_handler start
[ 9424.083718] sd 3:0:0:0: [sdb] tag#0 uas_zap_pending 0 uas-tag 1 inflight: CMD 
[ 9424.083725] sd 3:0:0:0: [sdb] tag#0 CDB: Read(10) 28 00 00 00 00 80 00 01 80 00
[ 9424.211955] usb 2-1: reset SuperSpeed USB device number 4 using xhci_hcd

```



lsusb
=====

```text
$ lsusb 
Bus 002 Device 005: ID 174c:55aa ASMedia Technology Inc. ASM1051E SATA 6Gb/s bridge, ASM1053E SATA 6Gb/s bridge, ASM1153 SATA 3Gb/s bridge
```
Windows 10's load  the "BayhubTech Intergate MMS/SD" driver for this device, but no linux driver is found on BayhubTech site.
Someone mentions a work around is "disabla UAS", from the dmesg it seems that the device doesn't response to specific UAS scsi command.


USB Attached SCSI (UAS) 
========================
 
UAS was introduced as part of the USB 3.0 standard, but can also be used with devices complying to the slower USB 2.0 standard.
UAS drivers generally provide faster transfers when compared to the older USB Mass Storage Bulk-Only Transport (BOT) protocol drivers
When used with an SSD, UAS is considerably faster than BOT for random reads and writes, but still well below the speed of a native SATA 3 interface (6 Gbit/s).


diable uas
==========

```text
the quirks parameter of  usb-storage has the following foramt:
"VID:PID:flags",
the vendor_id and product_id can be found in dmesg or lsusb output.

see usb_stor_adjust_quirks function in 
https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git/tree/drivers/usb/storage/usb.c


"options usb-storage quirks=Vendor_ID:Product_ID:u" in modprobe conf file,
or try to set it at runtime.

cat /sys/module/usb_storage/parameters/quirks 
sudo -i

echo "174c:55aa:u"  > /sys/module/usb_storage/parameters/quirks 


rmmod uas
rmmod usb_storage
modprobe usb_storage quirks=174c:55aa:u

```




dmesg  usb 3.0
==============
Still doesn't work on usb3.0 hub.
```text
May  3 14:14:12  kernel: [11667.505597] usb 2-1: new SuperSpeed USB device number 10 using xhci_hcd
May  3 14:14:12  kernel: [11667.526303] usb 2-1: New USB device found, idVendor=174c, idProduct=55aa
May  3 14:14:12  kernel: [11667.526310] usb 2-1: New USB device strings: Mfr=2, Product=3, SerialNumber=1
May  3 14:14:12  kernel: [11667.526314] usb 2-1: Product: KS-C7MG
May  3 14:14:12  kernel: [11667.526317] usb 2-1: Manufacturer: KINGSHARE
May  3 14:14:12  kernel: [11667.526321] usb 2-1: SerialNumber: 123456789010
May  3 14:14:12  kernel: [11667.527206] usb 2-1: UAS is blacklisted for this device, using usb-storage instead
May  3 14:14:12  kernel: [11667.527211] usb-storage 2-1:1.0: USB Mass Storage device detected
May  3 14:14:12  kernel: [11667.529328] usb-storage 2-1:1.0: Quirks match for vid 174c pid 55aa: c00000
May  3 14:14:12  kernel: [11667.529598] scsi host3: usb-storage 2-1:1.0
May  3 14:14:13  kernel: [11668.554187] scsi 3:0:0:0: Direct-Access     KINGSHAR KS-C7MG          0    PQ: 0 ANSI: 6
May  3 14:14:13  kernel: [11668.554815] sd 3:0:0:0: Attached scsi generic sg1 type 0
May  3 14:14:13  kernel: [11668.556061] sd 3:0:0:0: [sdb] 7865424 512-byte logical blocks: (4.03 GB/3.75 GiB)
May  3 14:14:13  kernel: [11668.556414] sd 3:0:0:0: [sdb] Write Protect is off
May  3 14:14:13  kernel: [11668.556419] sd 3:0:0:0: [sdb] Mode Sense: 43 00 00 00
May  3 14:14:13  kernel: [11668.556649] sd 3:0:0:0: [sdb] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
May  3 14:14:13  kernel: [11668.558606]  sdb: sdb1
May  3 14:14:13  kernel: [11668.559716] sd 3:0:0:0: [sdb] Attached SCSI disk
```


dmesg USB 2.0  
=============
No error was reported on USB2.0 hub.
```text
May  3 14:14:12  kernel: [11667.505597] usb 2-1: new SuperSpeed USB device number 10 using xhci_hcd
May  3 14:14:12  kernel: [11667.526303] usb 2-1: New USB device found, idVendor=174c, idProduct=55aa
May  3 14:14:12  kernel: [11667.526310] usb 2-1: New USB device strings: Mfr=2, Product=3, SerialNumber=1
May  3 14:14:12  kernel: [11667.526314] usb 2-1: Product: KS-C7MG
May  3 14:14:12  kernel: [11667.526317] usb 2-1: Manufacturer: KINGSHARE
May  3 14:14:12  kernel: [11667.526321] usb 2-1: SerialNumber: 123456xxxxxx
May  3 14:14:12  kernel: [11667.527206] usb 2-1: UAS is blacklisted for this device, using usb-storage instead
May  3 14:14:12  kernel: [11667.527211] usb-storage 2-1:1.0: USB Mass Storage device detected
May  3 14:14:12  kernel: [11667.529328] usb-storage 2-1:1.0: Quirks match for vid 174c pid 55aa: c00000
May  3 14:14:12  kernel: [11667.529598] scsi host3: usb-storage 2-1:1.0
May  3 14:14:13  kernel: [11668.554187] scsi 3:0:0:0: Direct-Access     KINGSHAR KS-C7MG          0    PQ: 0 ANSI: 6
May  3 14:14:13  kernel: [11668.554815] sd 3:0:0:0: Attached scsi generic sg1 type 0
May  3 14:14:13  kernel: [11668.556061] sd 3:0:0:0: [sdb] 7865424 512-byte logical blocks: (4.03 GB/3.75 GiB)
May  3 14:14:13  kernel: [11668.556414] sd 3:0:0:0: [sdb] Write Protect is off
May  3 14:14:13  kernel: [11668.556419] sd 3:0:0:0: [sdb] Mode Sense: 43 00 00 00
May  3 14:14:13  kernel: [11668.556649] sd 3:0:0:0: [sdb] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
May  3 14:14:13  kernel: [11668.558606]  sdb: sdb1
May  3 14:14:13  kernel: [11668.559716] sd 3:0:0:0: [sdb] Attached SCSI disk
```



IO Testing
==========
```text
# fdisk -l /dev/sdb
Disk /dev/sdb：3.8 GiB，4027097088 字节，7865424 个扇区
单元：扇区 / 1 * 512 = 512 字节
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x1b6ccbbc

设备       启动  起点    末尾    扇区  大小 Id 类型
/dev/sdb1        2048 7865423 7863376  3.8G 83 Linux


# parted /dev/sdb print
Model: KINGSHAR KS-C7MG (scsi)
磁盘 /dev/sdb: 4027MB
Sector size (logical/physical): 512B/512B
分区表：msdos
Disk Flags: 

数字  开始：  End     大小    类型     文件系统  标志
 1    1049kB  4027MB  4026MB  primary  ext3



root@:/home/# dd if=/dev/sdb of=disk_sdb_ext3.img bs=4k
记录了983178+0 的读入
记录了983178+0 的写出
4027097088 bytes (4.0 GB, 3.8 GiB) copied, 151.413 s, 26.6 MB/s
root@:/home/# 
```

