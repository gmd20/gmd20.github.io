来源
http://bbs.chinaunix.net/thread-4137351-1-1.html

很早之前试过用__nf_nat_mangle_tcp_packet来修改TCP包，好像修改是http的url。
但修改http包的网页内容时，长度改变之后，是还需要修改一下http协议里面对应的content-length和 chunk size才行。
chinaLinux的网友测试的看他那些写的结果是这样。图片来自该帖子，wireshark里面查看需要修改几个属性。
这个很邪恶的功能啊，以后的网络传输都用https协议吧。
