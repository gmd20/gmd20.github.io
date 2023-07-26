https://github.com/cilium/pwru
可以把满足“ tcpdump 抓包语法的”  网络包经过的 内核函数都打印出来，很直观的看到内核里面处理这个网络包的流程。

perf probe  利用kprobe可以 在任意函数的含行号上打印变量，可以定位到某个函数的执行流程吧。
```text
Usage: ./pwru [options] [pcap-filter]
    Availble pcap-filter: see "man 7 pcap-filter"
    Availble options:
      --all-kmods                 attach to all available kernel modules
      --backend string            Tracing backend('kprobe', 'kprobe-multi'). Will auto-detect if not specified.
      --filter-func string        filter kernel functions to be probed by name (exact match, supports RE2 regular expression)
      --filter-mark uint32        filter skb mark
      --filter-netns uint32       filter netns inode
      --filter-track-skb          trace a packet even if it does not match given filters (e.g., after NAT or tunnel decapsulation)
      --kernel-btf string         specify kernel BTF file
      --kmods strings             list of kernel modules names to attach to
      --output-file string        write traces to file
      --output-limit-lines uint   exit the program after the number of events has been received/printed
      --output-meta               print skb metadata
      --output-skb                print skb
      --output-stack              print stack
      --output-tuple              print L4 tuple
      --timestamp string          print timestamp per skb ("current", "relative", "absolute", "none") (default "none")
      --version                   show pwru version and exit

```

cloudflare的技术博客 分享的方法   
https://blog.cloudflare.com/lost-in-transit-debugging-dropped-packets-from-negative-header-lengths/
