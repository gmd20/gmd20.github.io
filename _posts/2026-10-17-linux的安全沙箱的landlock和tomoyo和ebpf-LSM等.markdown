Landlock , tomoyo, SELinux / AppArmor,  eBPF LSM 这些都属于 linux LSM 框架的
代码都在
https://elixir.bootlin.com/linux/v7.2.11/source/security

landlock 比较有意思， 说是很多 AI agent 都用这个来实现安全沙箱。
这个landlock 有syscall接口，一旦设置了 某些文件的限制后，就不可恢复了， root用户也不行。
https://docs.linuxkernel.org.cn/security/landlock.html 

tomoyo 好像也是按照文件路径设置访问权限的。
