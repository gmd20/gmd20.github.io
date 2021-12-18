iptables提供了log和trace这2个target，nlog是写内核日志，trace是更详细的规则执行步骤和各个情况。

log的使用的例子，打印dns包
```text

iptables -t mangle -i eth3 -I PREROUTING 1 -p udp -m udp --dport 53  -j LOG --log-prefix '==dns== '

iptables -t raw -A PREROUTING -p udp -m udp --dport 53 -j TRACE
xtables-nft-multi xtables-monitor -t
```
，
trace的使用的例子，只能添加在raw表， xtables-monitor命令会就看到规则的执行情况和匹配过程了。
```text

iptables -t raw -A PREROUTING -p udp -m udp --dport 53 -j TRACE
xtables-nft-multi xtables-monitor -t
```


redhat8 已经要用nft命令才行，因为有一些nft的rule在iptables命令里面是显示不出来的, 今天有一个问题排查了一天发现wireguard配置了一条特殊的nft规则。
```text
nft list ruleset
nft list tables
nft list table ip nat
```
