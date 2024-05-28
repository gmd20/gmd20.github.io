抓包里面有时想区分抓到的包是从本机发送出去的，还是从外面进来的。有的交换机（还是别的设备）好像会把本机发送出去的arp请求又大量重新发送回来，关靠源mac看不出。

这时tcpdump是有个参数指定抓包方向的。
```text
       -Q direction
       --direction=direction
              Choose send/receive direction direction for which packets should be captured. Possible values are `in',
              `out' and `inout'. Not available on all platforms.
```

记得packet socket是同时抓进出两个方向的，刚好代码里面也有需要这个功能，看一下 tcpdump怎么实现的。tcpdump底层应该用的libpcap。

```c
typedef enum {
       PCAP_D_INOUT = 0,
       PCAP_D_IN,
       PCAP_D_OUT
} pcap_direction_t;


/*
 * linux_check_direction()
 *
 * Do checks based on packet direction.
 */
static inline int
linux_check_direction(const pcap_t *handle, const struct sockaddr_ll *sll)
{
	struct pcap_linux	*handlep = handle->priv;

	if (sll->sll_pkttype == PACKET_OUTGOING) {
		/*
		 * Outgoing packet.
		 * If this is from the loopback device, reject it;
		 * we'll see the packet as an incoming packet as well,
		 * and we don't want to see it twice.
		 */
		if (sll->sll_ifindex == handlep->lo_ifindex)
			return 0;

		/*
		 * If this is an outgoing CAN or CAN FD frame, and
		 * the user doesn't only want outgoing packets,
		 * reject it; CAN devices and drivers, and the CAN
		 * stack, always arrange to loop back transmitted
		 * packets, so they also appear as incoming packets.
		 * We don't want duplicate packets, and we can't
		 * easily distinguish packets looped back by the CAN
		 * layer than those received by the CAN layer, so we
		 * eliminate this packet instead.
		 *
		 * We check whether this is a CAN or CAN FD frame
		 * by checking whether the device's hardware type
		 * is ARPHRD_CAN.
		 */
		if (sll->sll_hatype == ARPHRD_CAN &&
		     handle->direction != PCAP_D_OUT)
			return 0;

		/*
		 * If the user only wants incoming packets, reject it.
		 */
		if (handle->direction == PCAP_D_IN)
			return 0;
	} else {
		/*
		 * Incoming packet.
		 * If the user only wants outgoing packets, reject it.
		 */
		if (handle->direction == PCAP_D_OUT)
			return 0;
	}
	return 1;
}
```

看代码就是 “sll->sll_pkttype == PACKET_OUTGOING”了，看一下packet的文档 https://www.man7.org/linux/man-pages/man7/packet.7.html

```text
       SOCK_RAW packets are passed to and from the device driver without
       any changes in the packet data.  When receiving a packet, the
       address is still parsed and passed in a standard sockaddr_ll
       address structure.  When transmitting a packet, the user-supplied
       buffer should contain the physical-layer header.  That packet is
       then queued unmodified to the network driver of the interface
       defined by the destination address.  Some device drivers always
       add other headers.  SOCK_RAW is similar to but not compatible
       with the obsolete AF_INET/SOCK_PACKET of Linux 2.0.

       SOCK_DGRAM operates on a slightly higher level.  The physical
       header is removed before the packet is passed to the user.
       Packets sent through a SOCK_DGRAM packet socket get a suitable
       physical-layer header based on the information in the sockaddr_ll
       destination address before they are queued.

       By default, all packets of the specified protocol type are passed
       to a packet socket.  To get packets only from a specific
       interface use bind(2) specifying an address in a struct
       sockaddr_ll to bind the packet socket to an interface.  Fields
       used for binding are sll_family (should be AF_PACKET),
       sll_protocol, and sll_ifindex.

       The connect(2) operation is not supported on packet sockets.

       When the MSG_TRUNC flag is passed to recvmsg(2), recv(2), or
       recvfrom(2), the real length of the packet on the wire is always
       returned, even when it is longer than the buffer.

   Address types
       The sockaddr_ll structure is a device-independent physical-layer
       address.

           struct sockaddr_ll {
               unsigned short sll_family;   /* Always AF_PACKET */
               unsigned short sll_protocol; /* Physical-layer protocol */
               int            sll_ifindex;  /* Interface number */
               unsigned short sll_hatype;   /* ARP hardware type */
               unsigned char  sll_pkttype;  /* Packet type */
               unsigned char  sll_halen;    /* Length of address */
               unsigned char  sll_addr[8];  /* Physical-layer address */
           };

       The fields of this structure are as follows:

       sll_protocol
              is the standard ethernet protocol type in network byte
              order as defined in the <linux/if_ether.h> include file.
              It defaults to the socket's protocol.

       sll_ifindex
              is the interface index of the interface (see
              netdevice(7)); 0 matches any interface (only permitted for
              binding).  sll_hatype is an ARP type as defined in the
              <linux/if_arp.h> include file.

       sll_pkttype
              contains the packet type.  Valid types are PACKET_HOST for
              a packet addressed to the local host, PACKET_BROADCAST for
              a physical-layer broadcast packet, PACKET_MULTICAST for a
              packet sent to a physical-layer multicast address,
              PACKET_OTHERHOST for a packet to some other host that has
              been caught by a device driver in promiscuous mode, and
              PACKET_OUTGOING for a packet originating from the local
              host that is looped back to a packet socket.  These types
              make sense only for receiving.

       sll_addr
       sll_halen
              contain the physical-layer (e.g., IEEE 802.3) address and
              its length.  The exact interpretation depends on the
              device.

       When you send packets, it is enough to specify sll_family,
       sll_addr, sll_halen, sll_ifindex, and sll_protocol.  The other
       fields should be 0.  sll_hatype and sll_pkttype are set on
       received packets for your information.

```

