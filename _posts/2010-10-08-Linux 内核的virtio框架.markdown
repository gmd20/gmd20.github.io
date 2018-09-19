VirtIO 是一个用来实现“虚拟IO”的通用框架，典型的用于虚拟机上面实现虚拟的pci，网卡，磁盘等虚拟设备，kvm等都使用了这个技术。大概浏览了一下相关的内核代码，这个virtio应该是专门应用于那种“半虚拟化？（部分虚拟化）”的虚拟机的来虚拟各种通用io设备的，好像不是很适合单纯的用来虚拟设备。

参考资料

Virtio：针对 Linux 的 I/O 虚拟化框架 http://www.ibm.com/developerworks/cn/linux/l-virtio/index.html?ca=drs-cn-0304

KVM的virtio说明文档：

http://www.linux-kvm.org/page/Virtio

http://www.linux-kvm.org/wiki/images/d/dd/KvmForum2007%24kvm_pv_drv.pdf

内核相关示例源码有；

http://lxr.linux.no/linux+v2.6.35.7/drivers/net/virtio_net.c

http://lxr.linux.no/linux+v2.6.35.7/drivers/block/virtio_blk.c

http://lxr.linux.no/linux+v2.6.35.7/drivers/virtio/virtio.c

前段接口（guest端）的例子：

http://lxr.linux.no/linux+v2.6.35.7/drivers/s390/kvm/kvm_virtio.c

http://lxr.linux.no/linux+v2.6.35.7/drivers/lguest/lguest_device.c

如果想用virtio来实现一个虚拟设备大概流程是这样的:

1、自己实现scan设备的功能，然后调用register_virtio_device 函数来注册发现的虚拟设备。参考drivers/lguest/lguest_device.c 里面的scan_devices 函数。一个典型的实现是，虚拟机主机提供的共享内存配置，里面提供了虚拟设备的列表一样的配置，然后自己根据配置注册device。

      注：pci总线等应该也是要扫描各个插槽，然后发现新的pci设备，然后创建device结构，再调用到驱动的probe函数里面的，应该各个总线都有发现新设备的机制，device一般是scan的时候创建。

2、为virtio_device 设备准备好virtqueue 队列。参考drivers/s390/kvm/kvm_virtio.c 里面的kvm_find_vq 函数。特别的是要提供自己的virtqueue 结构的notify函数。notify函数用于通知host主机队列里面已经有消息存在了，一般是hypercall ，就是通过vmcall指令或者traps（中断？）来通知主机。例如这个kvm代码。


160/*
161 * When the virtio_ring code wants to notify the Host, it calls us here and we
162 * make a hypercall. We hand the address of the virtqueue so the Host
163 * knows which virtqueue we're talking about.
164 */
165static void kvm_notify(struct virtqueue *vq)
166{
167        struct kvm_vqconfig *config = vq->priv;
168
169        kvm_hypercall1(KVM_S390_VIRTIO_NOTIFY, config->address);
170}
171

http://lxr.linux.no/linux+v2.6.35.7/arch/x86/include/asm/kvm_para.h

69
70/* This instruction is vmcall. On non-VT architectures, it will generate a
71 * trap that we will then rewrite to the appropriate instruction.
72 */
73#define KVM_HYPERCALL ".byte 0x0f,0x01,0xc1"
74
75/* For KVM hypercalls, a three-byte sequence of either the vmrun or the vmmrun
76 * instruction. The hypervisor may replace it with something else but only the
77 * instructions are guaranteed to be supported.
78 *
79 * Up to four arguments may be passed in rbx, rcx, rdx, and rsi respectively.
80 * The hypercall number should be placed in rax and the return value will be
81 * placed in rax. No other registers will be clobbered unless explicited
82 * noted by the particular hypercall.
83 */
84
85static inline long kvm_hypercall0(unsigned int nr)
86{
87        long ret;
88        asm volatile(KVM_HYPERCALL
89                     : "=a"(ret)
90                     : "a"(nr)
91                     : "memory");
92        return ret;
93}
94
95static inline long kvm_hypercall1(unsigned int nr, unsigned long p1)
96{
97        long ret;
98        asm volatile(KVM_HYPERCALL
99                     : "=a"(ret)
100                     : "a"(nr), "b"(p1)
101                     : "memory");
102        return ret;
103}
104

3、应该/drivers/net/virtio_net.c 等代码已经实现了前端的网卡，磁盘等驱动了。所以只要你register_virtio_device注册了之后，客户机应该是可以看到相应的虚拟网卡的了。

这些驱动使用virtio的框架方法大概是这样的，

virtqueue_get_buf    这个函数用来获取virtqueue队列里面收到的消息。

virtqueue_add_buf 把发送到消息缓存写到virtqueue队列，然后 调用virtqueue_kick函数来通知host主机队列里面有消息需要它接受。 virtqueue_kick将调用到virtqueue结构的notify函数，notify函数通过中断或者高级的虚拟机指令mcall通知到host主机。 host主机，根据实现可能是共享内存的hypercall的，就可以直接复制相应的内存过去就可以了。可以去看相应的虚拟机的hypercall的实现代码。

virtio的作用就是提供一种居于virtqueue的通用的虚拟io框架。
