
nftables的sets比ipset iptables这些灵活一些，但感觉也复杂了一些。

```text
nft list ruleset
nft list tables
nft list table ip nat
nft flush table ip  example_table
nft delete table ip  example_table

nft add table inet example_table
nft add set inet example_table example_set '{ type ipv4_addr . ipv4_addr; size 65536; }'
nft add element inet example_table example_set '{ 172.16.0.1 . 10.0.0.1 }'
nft add chain inet example_table example_chain
nft add rule inet example_table example_chain  ip saddr . ip daddr '@example_set' drop
```

# 参考
https://www.netfilter.org/projects/nftables/manpage.html    
https://wiki.nftables.org/wiki-nftables/index.php/Sets   
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-using_sets_in_nftables_commands   
https://wiki.nftables.org/wiki-nftables/index.php/Moving_from_ipset_to_nftables   

