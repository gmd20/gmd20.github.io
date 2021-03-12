# netconsole

```text
modprobe netconsole netconsole=6666@192.168.1.2/eth1,6666@192.168.1.1/c8:5b:76:e4:7a:dd oops_only=1
nc -l -u 6666
echo c > /proc/sysrq-trigger
```
实际测试发现，netconsole在有些情况下保存不了完整的oops信息的，比如在网卡的软中断上奔溃了，可能这个netconsole依赖的网卡中断不正常的原因？
只会看到一行BUG，后面的关键信息都没有了。 这个没有串口线可靠。

如过想在linux上面用rsyslog接收，可以在/etc/rsyslog.d/目录创建配置文件
 ```text
 cat /etc/rsyslog.d/testnetconsole.conf
 $ModLoad imudp
$UDPServerRun 6666
$template DynFile,"/root/netconsole/%fromhost-ip%.log"
if $fromhost-ip startswith '172.16.' then ?DynFile
& ~
 ```
 然后/etc/init.d/rsyslog restart 重启，rsyslog就会监听6666端口，并把日志写到 "/root/netconsole“ 目录下了。
 这个netconsole不但是内核的日志，应用层的的syslog好像也发过来了，如果上面不指定 oops_only=1的话

# kdump
https://www.kernel.org/doc/Documentation/kdump/kdump.txt

kdump应该总是可靠的，不过用起来要配置好多东西的，看看redhat的封装。

grub里面内核启动参数要保留内存
```text
kernel crashkernel=auto
```

/etc/kdump.conf 配置里面制定最后的dump相关的设置
```text
core_collector  makedumpfile --dump-dmesg /proc/vmcore dmesgfile
```

kdump 这个service一共就是调用kdumpctl和kexec来加载内核吧。
```text
/usr/bin/kdumpctl start 
/usr/bin/kdumpctl stop
```

/sbin/mkdumprd  这个会创建kdump使用的initrd

```text
[root@localhost ming]# rpm -ql kexec-tools-2.0.15-13.el7.x86_64
/etc/kdump.conf
/etc/makedumpfile.conf.sample
/etc/sysconfig/kdump
/sbin/kdump
/sbin/kexec
/sbin/makedumpfile
/sbin/mkdumprd
/sbin/vmcore-dmesg
/usr/bin/kdumpctl
/usr/lib/dracut/modules.d/99kdumpbase
/usr/lib/dracut/modules.d/99kdumpbase/kdump-capture.service
/usr/lib/dracut/modules.d/99kdumpbase/kdump-emergency.service
/usr/lib/dracut/modules.d/99kdumpbase/kdump-emergency.target
/usr/lib/dracut/modules.d/99kdumpbase/kdump-error-handler.service
/usr/lib/dracut/modules.d/99kdumpbase/kdump-error-handler.sh
/usr/lib/dracut/modules.d/99kdumpbase/kdump.sh
/usr/lib/dracut/modules.d/99kdumpbase/module-setup.sh
/usr/lib/dracut/modules.d/99kdumpbase/monitor_dd_progress
/usr/lib/kdump
/usr/lib/kdump/kdump-lib-initramfs.sh
/usr/lib/kdump/kdump-lib.sh
/usr/lib/systemd/system-generators/kdump-dep-generator.sh
/usr/lib/systemd/system/kdump.service
/usr/lib/udev/rules.d
/usr/lib/udev/rules.d/98-kexec.rules
/usr/share/doc/kexec-tools-2.0.15
/usr/share/doc/kexec-tools-2.0.15/COPYING
/usr/share/doc/kexec-tools-2.0.15/News
/usr/share/doc/kexec-tools-2.0.15/TODO
/usr/share/doc/kexec-tools-2.0.15/fadump-howto.txt
/usr/share/doc/kexec-tools-2.0.15/kdump-in-cluster-environment.txt
/usr/share/doc/kexec-tools-2.0.15/kexec-kdump-howto.txt
/usr/share/doc/kexec-tools-2.0.15/supported-kdump-targets.txt
/usr/share/kdump
/usr/share/man/man5/kdump.conf.5.gz
/usr/share/man/man5/makedumpfile.conf.5.gz
/usr/share/man/man8/kdump.8.gz
/usr/share/man/man8/kdumpctl.8.gz
/usr/share/man/man8/kexec.8.gz
/usr/share/man/man8/makedumpfile.8.gz
/usr/share/man/man8/mkdumprd.8.gz
/usr/share/man/man8/vmcore-dmesg.8.gz
/var/crash
```


