```text

iptables -t raw -I PREROUTING 1 -p udp -m length --length 52 -m u32 --u32 "32 & 0xFFFFFFFF = 0x584D5359 && 44 & 0xFFFFFFFF= 0x464c5559" -j DROP

 length这个target可以匹配包长度， --length 为wireshark看到ip头的total length的数值
 u32 匹配udp包的内容， “偏移地址16”对应“目的ip地址”，其他偏移根据目的ip的偏移计算就行了，
 "32 & 0xFFFFFFFF = 0x584D5359“ 意思就是说第32个偏移的十六进制字节是0x584D5359

```
