
# facebook开源的 layer 4 负载均衡程序。基于XDP
https://github.com/facebookincubator/katran
```text
# katran

[![Build Status](https://travis-ci.org/facebookincubator/katran.svg?branch=master)](https://travis-ci.org/facebookincubator/katran)

Katran is a C++ library and [`BPF`](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter) program to build high-performance
layer 4 load balancing forwarding plane. Katran leverages [`XDP infrastructure`](https://www.iovisor.org/technology/xdp)
from the kernel to provide an in-kernel facility for fast packet's processing.

### Katran's key features:

1. Blazing fast (especially w/ XDP in driver mode).
2. Performance scaling linearly with a number of NIC's RX queues.
3. RSS friendly encapsulation.

### katran's features description:

1. __Fast :__ katran uses XDP for packet forwarding, which allows
to run packet handling routines right after packet has been received by
network interface card (NIC) and before kernel had any chance to run (when XDP
is working in "driver mode", katran supports "generic XDP" mode of operation
as well (with some performance degradation compare to "driver mode")).

2. __Performance scales linearly with a number of NIC's RX queues :__ The way XDP
works is that it invokes BPF program on every received packet, and if your
NIC has multiple queues, for each of them BPF program will be invoked
independently. As katran is completely lockless and uses per-cpu
versions of BPF maps - it scales linearly.

3. __RSS friendly encapsulation :__ katran is uses ipip encapsulation for packet
forwarding from L4 lb to L7 lb. However, to be able to work in conjunction with
RSS on L7 lb receive side, instead of using the same source for every ipip
packet, katran crafts a special one, in such a way, that different flows will have
different outer (ipip) source IP, but packets in same flow will always have
the same.

4. Fixed size (size is configurable on start) connection tracking table w/ LRU
strategy for eviction of old entries.

5. __Modified Maglev hashing for connections :__ It provides us good resiliency in
case of failure and excellent load balancing features. The hashing was modified
to be able to support unequal weights for backend (L7 lbs) servers

6. __No need for busylooping on receive path :__ Your load balancer will barely
consume any CPU if there is no traffic to serve.

7. katran (and XDP in general) allows you to run any application w/o any
performance penalties on the same server (compare to some of other
"kernel bypass" technologies)
```

# XDP 被合并到linux kernel (4.13+)了
应该是可以利用内核ebpf虚拟机，直接在网络包没有进入linux内核队列时，就可以丢包进行过滤和修改。很适合用于做ddos过滤还有，简单的修改包头做负载均衡等。
看说明这个性能可以媲美DPDK，不过这个可以和linux的协议栈一起使用。很有意思的东西。
https://www.iovisor.org/technology/xdp
```text
XDP or eXpress Data Path provides a high performance, programmable network data path in the Linux kernel as part of the IO Visor Project. XDP provides bare metal packet processing at the lowest point in the software stack which makes it ideal for speed without compromising programmability. Furthermore, new functions can be implemented dynamically with the integrated fast path without kernel modification. Other key benefits of XDP includes the following:

It does not require any specialized hardware
It does not required kernel bypass
It does not replace the TCP/IP stack
It works in concert with TCP/IP stack along with all the benefits of BPF

Use Cases
---------
Use cases for XDP include the following:

Pre-stack processing like filtering to support DDoS mitigation
Forwarding and load balancing
Batching techniques such as in Generic Receive Offload
Flow sampling, monitoring
ULP processing (i.e. message delineation)


XDP and DPDK
------------
XDP is sometimes juxtaposed with DPDK when both are perfectly fine approaches. XDP offers another option for users who want performance while still leveraging the programmability of the kernel. Some of the functions that XDP delivers include the following:

Removes the need for 3rd party code and licensing
Allows option of busy polling or interrupt driven networking
Removes the need to allocate large pages
Removes the need for dedicated CPUs as users have more options on structuring work between CPUs
Removes the need to inject packets into the kernel from a 3rd party user space application
Removes the need to define a new security model for accessing networking hardware
```


# cloudflare/bpftools
BPF代码生成器，用来过滤dns的ddos攻击，iptables竟然支持bpf字节代码作为过滤表达式的！！
https://github.com/cloudflare/bpftools
