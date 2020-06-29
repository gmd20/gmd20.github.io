# dropwatch

```text
yum install dropwatch
dropwatch -l kas
dropwatch> start

stap --poison-cache -vvv -L 'kernel.trace("*")'
stap /usr/share/systemtap/examples/network/dropwatch.stp  -c "sleep 100"

perf top -e skb:kfree_skb
```
