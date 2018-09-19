        以前还在上家公司做vmware esxi5 hypervisor 移植的时候，有接触到Linux网卡驱动看到这个多队列的，但不是很理解这个来用来干什么 ，那是也是和关系。看了这下面这篇文章才知道这个multiqueue原来是这样用的。按照文章说法，多队列之后，在多cpu机器上就可以每个cpu分配一个队列了，这个没个cpu都有独立的queue，“锁争用"和内存争用的情况就可以避免了，性能有很大提高，文章里面应用于 xen 的虚拟网卡，每个虚拟机有独立的一个队列，性能有很大的提高。那个hypercall的处理就可以减少很大工作量吧。文章都是2008年的了。

Network Virtualization: Breaking the Performance Barrier
http://queue.acm.org/detail.cfm?id=1348592

 

是在微博上看到 EMC 文件，说 一个前vmware的一个架构师的”虚拟设备虚拟io“方面的文章。 上面那个是这篇的推荐文章里面的一篇.

这篇就是个概述吧，讲的很虚，不过提到了很多设计方面的或者性能方面的问题。可以去看一下。作者应该都是比较有水平的吧，以前看vmware hypervisor那些代码，全部底层都自己写的，然后上层模拟一下Linux的接口，然后上面跑linux 的驱动，代码写的确实工整。exsi开源的可以去学习一下吧。

Decoupling a logical device from its physical implementation offers many compelling advantages

By CaRL waLDsPuRGeR anD MenDeL RosenBLuM

i/o Virtualization

http://delivery.acm.org/10.1145/2070000/2063194/p66-waldspurger.pdf?ip=113.119.221.63&acc=OPEN&CFID=68511132&CFTOKEN=62863101&__acm__=1330619292_7991c0dea4be09488ad3697d563d50d8

 
