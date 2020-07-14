cloudflare的blog这篇Sandboxing in Linux with zero lines of code
https://blog.cloudflare.com/sandboxing-in-linux-with-zero-lines-of-code/


提到linux 一个很有意思的syscall， 可以用来设置进程 syscall的白名单列表。就是可以设置一个BPF来显示各种syscall调用吧。 现在好像到处是BPF的身影。
这个调用还是很有意思的，systemd好像也支持 配置允许执行syscall的方式了运行进程，不过cloudflare的人做的更彻底吧。前几天刚好想linux怎么实现sandbox，这个也是一个思路吧。

Seccomp BPF (SECure COMPuting with filters)
https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html
