好像最近open vswitch改进很多，搞了一个OVN（Open Virtual Network ），现在也支持 connection tracking 和 NAT功能了。
那stateful firewall就可以实现了，应该是可以抛iptables了吧。
OVN还是比较有意思的，已经支持L3 router功能了，不需要linux内核的ip协议栈的参与了， 集成dhcp 还有arp过滤等功能。
看上去应该是可以取代linux的 “软路由”功能了，这个东西如果配合ovs-dpdk的datapath来使用，那性能是要比普通的linux协议栈的
NAT网关实现性能要好很多吧。 看评测 ovs-dpdk的性能比非dpdk的要十几二十倍都有。而且基于ovs的firewall和coonn tracking的
性能比linux内核ip协议栈+iptables扩展性要好的。

就是这个OVN出来没多久，文档比较欠缺一些。而且还在持续的开发过程中，可能有的功能还不是很完善。看了一下文档，感觉操作好复杂。
还不是很理解各种概念，有时间再学习一下。

ovn-architecture - Open Virtual Network architecture
http://openvswitch.org/support/dist-docs/ovn-architecture.7.txt



