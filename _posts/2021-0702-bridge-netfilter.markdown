http://ebtables.netfilter.org/documentation/bridge-nf.html   
http://ebtables.netfilter.org/misc/brnf-faq.html   

bridge是二层的，本来是使用ebtables这种二层过滤的， 
bridge-netfilter模块可以在网桥上执行iptables的包过滤

一些控制开关：
/proc/sys/net/bridge/bridge-nf-call-iptables   
/sys/devices/virtual/net/<bridge-name>/bridge/nf_call_iptables

modinfo bridge   
modinfo br_netfilter   
https://elixir.bootlin.com/linux/latest/source/net/bridge/br_if.c   

