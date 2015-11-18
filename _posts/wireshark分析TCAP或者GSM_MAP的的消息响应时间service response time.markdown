在wireshark里面默认为每个http response 消息统计http.time, 这样就是在 statistics 里面io graph 界面的 advance 功能里面
查看   响应时间的 按照时间分布曲线了。  一个不错的功能。


TCAP协议也可以配置一下，它也会统计每个消息request 和response 的响应时间的。
在TCAP协议设置选项里面，勾上  “service response time analyse” 和“persistent stat of SRT” 。


但发现包信息的STAT 属性下面 还是没出现 session time相关的标签。
看上去是 tcap 的begin 和 tcap end 消息的hash 匹配不上。
需要在SCCP协议选项里面勾上， “set source and destination GT addresses”。 
这个影响TCAP的begin 和end的消息hash比较。 改了之后begin和end消息匹配正常。 就有 session time的显示了。
tcap.srt.sessiontime 这个时间就可以用于statistics里面io graph 功能里面的响应时间统计了。



一开始还以为要自己用lua或者c来写个额外的插件，但tcap内部自己就有统计了。
https://www.wireshark.org/docs/wsdg_html_chunked/PartDevelopment.html
https://www.wireshark.org/docs/wsdg_html_chunked/lua_module_Tree.html
https://wiki.wireshark.org/Lua

分析tcap协议，计算这个serice response time的对应源码在
wireshark-1.12.1\epan\dissectors\packet-tcap.c
