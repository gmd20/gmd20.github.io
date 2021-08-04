http://ebtables.netfilter.org/documentation/bridge-nf.html   
http://ebtables.netfilter.org/misc/brnf-faq.html   

bridge是二层的，本来是使用ebtables这种二层过滤的， 
bridge-netfilter模块可以在网桥上执行iptables的包过滤

# 一些控制开关：
/proc/sys/net/bridge/bridge-nf-call-iptables   
/sys/devices/virtual/net/br0/bridge/nf_call_iptables

modinfo bridge   
modinfo br_netfilter   
https://elixir.bootlin.com/linux/latest/source/net/bridge/br_if.c   
https://elixir.bootlin.com/linux/latest/source/net/bridge/br_fdb.c   # 交换机mac 表的实现   
https://elixir.bootlin.com/linux/latest/source/include/linux/rhashtable.h     
https://elixir.bootlin.com/linux/latest/source/net/bridge/br_stp.c   # stp的实现   
https://elixir.bootlin.com/linux/latest/source/net/bridge/br_vlan.c  # vlan的实现，有sysfs有选项控制是否打开vlan的支持, br_handle_vlan函数
  https://elixir.bootlin.com/linux/latest/source/net/bridge/netfilter/nf_conntrack_bridge.c  # 新内核里面有另外一个bridge的 conntack实现？
  
# 应用层管理、配置工具：
```text 
yum install bridge-utils  # brctl
  

brctl delbr br0
brctl addbr br0
brctl addif br0 eth0
brctl addif br0 eth1

brctl showmacs <brname> shows a list of learned MAC addresses for this bridge.
brctl setageing <brname> <time> 
brctl setgcint <brname> <time>  
brctl stp <bridge> <state>  #  If <state> is "on" or "yes" the STP will be turned on
brctl setbridgeprio <bridge> <priority> 
brctl setfd <bridge> <time> sets the bridge's 'bridge forward delay' to <time> seconds.
brctl sethello <bridge> <time> sets the bridge's 'bridge hello time' to <time> seconds.
brctl setmaxage <bridge> <time>
brctl setpathcost <bridge> <port> <cost>
brctl setportprio <bridge> <port> <priority> '''

  
  # iproute2包里面的 bridge
       bridge link set dev DEV [ cost COST ] [ priority PRIO ] [ state STATE ] [ guard { on | off
               } ] [ hairpin { on | off } ] [ fastleave { on | off } ] [ root_block { on | off }
               ] [ learning { on | off } ] [ learning_sync { on | off } ] [ flood { on | off } ]
               [ hwmode { vepa | veb } ] [ mcast_flood { on | off } ] [ mcast_to_unicast { on |
               off } ] [ neigh_suppress { on | off } ] [ vlan_tunnel { on | off } ] [ isolated {
               on | off } ] [ backup_port DEVICE ] [ nobackup_port ] [ self ] [ master ]

       bridge link [ show ] [ dev DEV ]

       bridge fdb { add | append | del | replace } LLADDR dev DEV { local | static | dynamic } [
               self ] [ master ] [ router ] [ use ] [ extern_learn ] [ sticky ] [ src_vni VNI ] {
               [ dst IPADDR ] [ vni VNI ] [ port PORT ] [ via DEVICE ] | nhid NHID }

       bridge fdb [ [ show ] [ br BRDEV ] [ brport DEV ] [ vlan VID ] [ state STATE ] [ dynamic ]

/ # bridge vlan help
Usage: bridge vlan { add | del } vid VLAN_ID dev DEV [ tunnel_info id TUNNEL_ID ]
                                                     [ pvid ] [ untagged ]
                                                     [ self ] [ master ]


hairpin是说是否允许原路返回的
echo 1 > /sys/class/net/br0/brif/eth1/hairpin_mode

```
