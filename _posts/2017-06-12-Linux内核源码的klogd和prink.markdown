
klogd读取的/proc/kmsg文件
http://elixir.free-electrons.com/linux/v4.12-rc5/source/fs/proc/kmsg.c

读取这个/proc/kmsg文件其实就是调用 syslog这个系统调用 
do_syslog
http://elixir.free-electrons.com/linux/v4.12-rc5/source/kernel/printk/printk.c#L1430

这个会等在一个锁上面


printk 会在执行完之后，调用console_unlock  wake_up_klogd函数唤醒这个锁。


/dev/kmsg 设备文件的操作 也子啊printk.c 里面 
```c
const struct file_operations kmsg_fops = {
	.open = devkmsg_open,
	.read = devkmsg_read,
	.write_iter = devkmsg_write,
	.llseek = devkmsg_llseek,
	.poll = devkmsg_poll,
	.release = devkmsg_release,
};
```

devkmsg_read   同样阻塞在 log_wait 这个锁上面

不过试了一下 cat /proc/kmsg 和 cat /dev/kmsg，前者只会阻塞等待新消息，  后者会把历史buf的消息也显示出来。
用strace 跟踪了一下 dmesg命令 应该是使用的/dev/kmsg文件来的吧。

其他的设置终端console的printk级别
echo 8 > /proc/sys/kernel/printk
