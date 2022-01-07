需要用clang把c语言写的代码编译成ebpf的object文件，然后用工具把object文件加载到内核。
不过也有纯golang的库用于编译和调试ebpf的  https://github.com/cilium/ebpf   

也有很多辅助工具用于加载bpf代码，比如https://github.com/iovisor/bcc 用于调试分析系统，已经集成了很多脚本了吧，好像可以取代systemtap.





BPF and XDP Reference Guide   
https://docs.cilium.io/en/stable/bpf/     
详细介绍了llvm编译bpf和ip 命令加载等用法, 内核提供 bpftool 命令用于辅助调试bpf代码   


bpf相关的资讯博客   
https://ebpf.io/   


Load XDP programs using the ip (iproute2) command   
https://fntlnz.wtf/post/xdp-ip-iproute/   
ip link的帮助手册 详细介绍了怎么用ip命令加载xdp代码   
https://www.man7.org/linux/man-pages/man8/ip-link.8.html   


WRITING AN XDP NETWORK FILTER WITH EBPF   
https://duo.com/labs/tech-notes/writing-an-xdp-network-filter-with-ebpf


简单的防火墙过滤ip黑名单的例子
https://github.com/gamemann/XDP-Firewall   
https://github.com/cppcoffee/ipblock   
https://github.com/massoudasadi/packiffer/blob/master/xdp_block_address.c   

redhat提供一个xdp-tools 工具包，有提供一个xdp-filter命令用户做xdp防火墙过滤。   
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/using-xdp-filter-for-high-performance-traffic-filtering-to-prevent-ddos-attacks_configuring-and-managing-networking   
https://www.redhat.com/en/blog/capturing-network-traffic-express-data-path-xdp-environment


之前记录的相关资料，bpf的防火墙应用:
BPF comes to firewalls   
https://lwn.net/Articles/747551/     

eBPF / XDP firewall and packet filtering   
http://vger.kernel.org/lpc_net2018_talks/ebpf-firewall-LPC.pdf   

L4Drop: XDP DDoS Mitigations   
https://blog.cloudflare.com/l4drop-xdp-ebpf-based-ddos-mitigations/   

https://github.com/gmd20/gmd20.github.io/blob/master/_posts/2019-05-01-XDP%E5%92%8CAF_XDP%E5%92%8Cbpfilter.markdown
