https://tools.ietf.org/html/rfc7011

现在的交换机路由器都支持通过Netflow或者sFlow协议实时导出 tcp，udp的连接信息给后端的监控服务器，
后端服务器好像一个通过udp来接收发过来的流信息进行分析，就可以 实时的最网络流量做监控或者统计了。

http://openvswitch.org/features/
https://github.com/openvswitch/ovs/blob/4d6f69df54b7d6ec2956875c683a9564cb175662/ofproto/ofproto-dpif-ipfix.c

IETF 组织在Netflow上面定义出了 IPFIX规范，  IPFIX 应该可以支持自定义扩展的，比如可以用来到处HTTP的URL这些信息。 
https://www.plixer.com/blog/netflow/exporting-urls-in-ipfix-not-netflow/

这个接口比较有意思，其实就是一个网络日志了，但还有流量等各种统计在里面了，还可以自定义。现在网上应该有很多所谓的“netflow analyzer” 可以分析这个日志吧。 cisco 的NBAR 技术也是利用这个接口，看思科的文档，好像他的NBAR2的宫娥能很强大，可以支持7层协议的识别。


ntop好像是个开源的工具，可以统计网络利用情况，有几个nDPI深度包分析等模块，支持IPFIX协议这些。
https://github.com/ntop  



openvswitch是支持netflow  sflow和这个IPFIX协议的。源码在这里
https://github.com/openvswitch/ovs/blob/4d6f69df54b7d6ec2956875c683a9564cb175662/ofproto/ofproto-dpif-ipfix.c

有时间再学习一下。
