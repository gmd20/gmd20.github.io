rtnetlink
=========
https://www.man7.org/linux/man-pages/man7/rtnetlink.7.html  
直接使用netlink应该比较难用


libnetlink
==========
https://www.man7.org/linux/man-pages/man3/libnetlink.3.html         
ip route get命令使用应该是libnetlink接口，不过centos-8好像没找到libnetlink.so 和libnetlink-devel包
需要直接iproute源码里面的 libnetlink.c  libnetlink.h 出来使用吧， 可以参考一下ip命令的代码 iproute.c。

比如参考他源码改动一个获取ip 路由的例子  route_get.c 
```c
#include <stdio.h>
#include <stdlib.h>
#include <asm/types.h>
#include <libnetlink.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>
#include <net/if.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>


const char *format_host(int af, int len, const void *addr)
{
	static char buf[256];
	switch (af) {
	case AF_INET:
	case AF_INET6:
		return inet_ntop(af, addr, buf, sizeof(buf));
		break;
	default:
		return "";
	}
}

int print_route(struct nlmsghdr *n, void *arg)
{
	FILE *fp = (FILE *)arg;
	struct rtmsg *r = NLMSG_DATA(n);
	int len = n->nlmsg_len;
	struct rtattr *tb[RTA_MAX+1];
	int ret;

	if (n->nlmsg_type != RTM_NEWROUTE && n->nlmsg_type != RTM_DELROUTE) {
		fprintf(stderr, "Not a route: %08x %08x %08x\n",
				n->nlmsg_len, n->nlmsg_type, n->nlmsg_flags);
		return -1;
	}

	len -= NLMSG_LENGTH(sizeof(*r));
	if (len < 0) {
		fprintf(stderr, "BUG: wrong nlmsg len %d\n", len);
		return -1;
	}

	parse_rtattr(tb, RTA_MAX, RTM_RTA(r), len);

	if (tb[RTA_GATEWAY]) {
		const struct rtattr *rta = tb[RTA_GATEWAY];
		int af = r->rtm_family;
		int len = RTA_PAYLOAD(rta);
		const void *addr = RTA_DATA(rta);
		const char *s = format_host(af, len, addr);
		fprintf(fp, "gateway : %s\n", s);
	}

	if (tb[RTA_VIA]) {
		const struct rtattr *rta = tb[RTA_VIA];
		size_t len = RTA_PAYLOAD(rta) - 2;
		const struct rtvia *via = RTA_DATA(rta);
		const char *s = format_host(via->rtvia_family, len, via->rtvia_addr);
		fprintf(fp, "via : %s\n", s);
	}

	if (tb[RTA_OIF]) {
		const struct rtattr *rta = tb[RTA_OIF];
		int ifindex = rta_getattr_u32(rta);
		char ifname[IF_NAMESIZE];
		if (if_indextoname(ifindex, ifname) != NULL) {
			fprintf(fp, "dev : %s\n", ifname);
		}
	}

	fflush(fp);
	return 0;
}


static int iproute_get(char *ipv4_addr)
{
	struct rtnl_handle rth;
	struct {
		struct nlmsghdr	n;
		struct rtmsg		r;
		char			buf[1024];
	} req = {
		.n.nlmsg_len = NLMSG_LENGTH(sizeof(struct rtmsg)),
		.n.nlmsg_flags = NLM_F_REQUEST,
		.n.nlmsg_type = RTM_GETROUTE,
		.r.rtm_family = AF_INET,
	};
	struct nlmsghdr *answer;
	unsigned char addr[sizeof(struct in6_addr)];

	if (inet_pton(AF_INET, ipv4_addr, addr) < 0) {
		fprintf(stderr, "Invalid ip address\n");
		return -1;
	}

	if (rtnl_open(&rth, 0) < 0) {
		fprintf(stderr, "Cannot open rtnetlink\n");
		return -1;
	}

	addattr_l(&req.n, sizeof(req), RTA_DST, &addr, 4);
	req.r.rtm_dst_len = 32; // subnet netmask
	// addattr32(&req.n, sizeof(req), RTA_IIF, 1);

	if (req.r.rtm_family == AF_UNSPEC)
		req.r.rtm_family = AF_INET;

	/* Only IPv4 supports the RTM_F_LOOKUP_TABLE flag */
	if (req.r.rtm_family == AF_INET)
		req.r.rtm_flags |= RTM_F_LOOKUP_TABLE;

	if (rtnl_talk(&rth, &req.n, &answer) < 0) {
		fprintf(stderr, "rtnl talk error\n");
		return -2;
	}

	if (print_route(answer, (void *)stdout) < 0) {
		fprintf(stderr, "An error :-)\n");
		free(answer);
		return -3;
	}

	rtnl_close(&rth);
	free(answer);
	return 0;
}

void main(int argc, char *argv[])
{
	if (argc > 1) {
		iproute_get(argv[1]);
	}
}

```
```text
gcc -I./ route_get.c libnetlink.c
```


libmnl
======
不过libnetlink应该还是比较难用的，所以它man page上面建议用netfilter开源的 libmnl。

https://netfilter.org/projects/libmnl/doxygen/html/index.html
它源码里面提供的rtnetlink的例子
http://git.netfilter.org/libmnl/tree/examples/rtnl?h=libmnl-1.0.4


参考他的代码写一个查询路由的代码
```c
/* This example is placed in the public domain. */
#include <netinet/in.h>
#include <arpa/inet.h>
#include <time.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <strings.h>
#include <net/if.h>

#include <libmnl/libmnl.h>
#include <linux/if_link.h>
#include <linux/rtnetlink.h>

static int data_attr_cb2(const struct nlattr *attr, void *data)
{
	/* skip unsupported attribute in user-space */
	if (mnl_attr_type_valid(attr, RTAX_MAX) < 0)
		return MNL_CB_OK;

	if (mnl_attr_validate(attr, MNL_TYPE_U32) < 0) {
		perror("mnl_attr_validate");
		return MNL_CB_ERROR;
	}
	return MNL_CB_OK;
}

static void attributes_show_ipv4(struct nlattr *tb[])
{
	if (tb[RTA_TABLE]) {
		printf("table=%u ", mnl_attr_get_u32(tb[RTA_TABLE]));
	}
	if (tb[RTA_DST]) {
		struct in_addr *addr = mnl_attr_get_payload(tb[RTA_DST]);
		printf("dst=%s ", inet_ntoa(*addr));
	}
	if (tb[RTA_SRC]) {
		struct in_addr *addr = mnl_attr_get_payload(tb[RTA_SRC]);
		printf("src=%s ", inet_ntoa(*addr));
	}
	if (tb[RTA_OIF]) {
		printf("oif=%u ", mnl_attr_get_u32(tb[RTA_OIF]));
	}
	if (tb[RTA_FLOW]) {
		printf("flow=%u ", mnl_attr_get_u32(tb[RTA_FLOW]));
	}
	if (tb[RTA_PREFSRC]) {
		struct in_addr *addr = mnl_attr_get_payload(tb[RTA_PREFSRC]);
		printf("prefsrc=%s ", inet_ntoa(*addr));
	}
	if (tb[RTA_GATEWAY]) {
		struct in_addr *addr = mnl_attr_get_payload(tb[RTA_GATEWAY]);
		printf("gw=%s ", inet_ntoa(*addr));
	}
	if (tb[RTA_METRICS]) {
		int i;
		struct nlattr *tbx[RTAX_MAX+1] = {};

		mnl_attr_parse_nested(tb[RTA_METRICS], data_attr_cb2, tbx);

		for (i=0; i<RTAX_MAX; i++) {
			if (tbx[i]) {
				printf("metrics[%d]=%u ", i, mnl_attr_get_u32(tbx[i]));
			}
		}
	}
	printf("\n");
}

static int data_attr_cb(const struct nlattr *attr, void *data)
{
	const struct nlattr **tb = data;
	int type = mnl_attr_get_type(attr);

	/* skip unsupported attribute in user-space */
	if (mnl_attr_type_valid(attr, RTA_MAX) < 0)
		return MNL_CB_OK;

	switch(type) {
	 case RTA_TABLE:
	 case RTA_DST:
	 case RTA_SRC:
	 case RTA_OIF:
	 case RTA_FLOW:
	 case RTA_PREFSRC:
	 case RTA_GATEWAY:
		 if (mnl_attr_validate(attr, MNL_TYPE_U32) < 0) {
			 perror("mnl_attr_validate");
			 return MNL_CB_ERROR;
		 }
		 break;
	 case RTA_METRICS:
		 if (mnl_attr_validate(attr, MNL_TYPE_NESTED) < 0) {
			 perror("mnl_attr_validate");
			 return MNL_CB_ERROR;
		 }
		 break;
	}
	tb[type] = attr;
	return MNL_CB_OK;
}

static int data_cb(const struct nlmsghdr *nlh, void *data)
{
	struct nlattr *tb[RTA_MAX+1] = {};
	struct rtmsg *rm = mnl_nlmsg_get_payload(nlh);

	/* protocol family = AF_INET | AF_INET6 */
	printf("family=%u ", rm->rtm_family);

	/* destination CIDR, eg. 24 or 32 for IPv4 */
	printf("dst_len=%u ", rm->rtm_dst_len);

	/* source CIDR */
	printf("src_len=%u ", rm->rtm_src_len);

	/* type of service (TOS), eg. 0 */
	printf("tos=%u ", rm->rtm_tos);

	/* table id:
	 *	RT_TABLE_UNSPEC		= 0
	 *
	 * 	... user defined values ...
	 *
	 *	RT_TABLE_COMPAT		= 252
	 *	RT_TABLE_DEFAULT	= 253
	 *	RT_TABLE_MAIN		= 254
	 *	RT_TABLE_LOCAL		= 255
	 *	RT_TABLE_MAX		= 0xFFFFFFFF
	 *
	 * Synonimous attribute: RTA_TABLE.
	 */
	printf("table=%u ", rm->rtm_table);

	/* type:
	 * 	RTN_UNSPEC	= 0
	 * 	RTN_UNICAST	= 1
	 * 	RTN_LOCAL	= 2
	 * 	RTN_BROADCAST	= 3
	 *	RTN_ANYCAST	= 4
	 *	RTN_MULTICAST	= 5
	 *	RTN_BLACKHOLE	= 6
	 *	RTN_UNREACHABLE	= 7
	 *	RTN_PROHIBIT	= 8
	 *	RTN_THROW	= 9
	 *	RTN_NAT		= 10
	 *	RTN_XRESOLVE	= 11
	 *	__RTN_MAX	= 12
	 */
	printf("type=%u ", rm->rtm_type);

	/* scope:
	 * 	RT_SCOPE_UNIVERSE	= 0   : everywhere in the universe
	 *
	 *      ... user defined values ...
	 *
	 * 	RT_SCOPE_SITE		= 200
	 * 	RT_SCOPE_LINK		= 253 : destination attached to link
	 * 	RT_SCOPE_HOST		= 254 : local address
	 * 	RT_SCOPE_NOWHERE	= 255 : not existing destination
	 */
	printf("scope=%u ", rm->rtm_scope);

	/* protocol:
	 * 	RTPROT_UNSPEC	= 0
	 * 	RTPROT_REDIRECT = 1
	 * 	RTPROT_KERNEL	= 2 : route installed by kernel
	 * 	RTPROT_BOOT	= 3 : route installed during boot
	 * 	RTPROT_STATIC	= 4 : route installed by administrator
	 *
	 * Values >= RTPROT_STATIC are not interpreted by kernel, they are
	 * just user-defined.
	 */
	printf("proto=%u ", rm->rtm_protocol);

	/* flags:
	 * 	RTM_F_NOTIFY	= 0x100: notify user of route change
	 * 	RTM_F_CLONED	= 0x200: this route is cloned
	 * 	RTM_F_EQUALIZE	= 0x400: Multipath equalizer: NI
	 * 	RTM_F_PREFIX	= 0x800: Prefix addresses
	 */
	printf("flags=%x\n", rm->rtm_flags);

	mnl_attr_parse(nlh, sizeof(*rm), data_attr_cb, tb);

	switch(rm->rtm_family) {
	 case AF_INET:
		 attributes_show_ipv4(tb);
		 break;
	}

	return MNL_CB_OK;
}

int main(int argc, char *argv[])
{
	if (argc <= 2) {
		printf("Usage: %s destination cidr\n", argv[0]);
		printf("Example: %s 10.0.1.12 32\n", argv[0]);
		exit(EXIT_FAILURE);
	}

	in_addr_t dst;
	if (!inet_pton(AF_INET, argv[1], &dst)) {
		printf("Bad destination\n");
		exit(EXIT_FAILURE);
	}

	uint32_t mask;
	if (sscanf(argv[2], "%u", &mask) == 0) {
		printf("Bad CIDR\n");
		exit(EXIT_FAILURE);
	}

	struct mnl_socket *nl;
	char buf[MNL_SOCKET_BUFFER_SIZE];
	struct nlmsghdr *nlh;
	struct rtmsg *rtm;
	int ret;
	unsigned int seq, portid;

	nlh = mnl_nlmsg_put_header(buf);
	nlh->nlmsg_type	= RTM_GETROUTE;
	nlh->nlmsg_flags = NLM_F_REQUEST;
	nlh->nlmsg_seq = seq = time(NULL);

	rtm = mnl_nlmsg_put_extra_header(nlh, sizeof(struct rtmsg));
	rtm->rtm_family = AF_INET;
	rtm->rtm_dst_len = mask;
	rtm->rtm_src_len = 0;

	mnl_attr_put_u32(nlh, RTA_DST, dst);

	nl = mnl_socket_open(NETLINK_ROUTE);
	if (nl == NULL) {
		perror("mnl_socket_open");
		exit(EXIT_FAILURE);
	}

	if (mnl_socket_bind(nl, 0, MNL_SOCKET_AUTOPID) < 0) {
		perror("mnl_socket_bind");
		exit(EXIT_FAILURE);
	}

	if (mnl_socket_sendto(nl, nlh, nlh->nlmsg_len) < 0) {
		perror("mnl_socket_send");
		exit(EXIT_FAILURE);
	}

	ret = mnl_socket_recvfrom(nl, buf, sizeof(buf));
	while (ret > 0) {
		ret = mnl_cb_run(buf, ret, seq, portid, data_cb, NULL);
		if (ret <= MNL_CB_STOP)
			break;

		if (ret == MNL_CB_OK) {
      // next query
      break;
    }
		ret = mnl_socket_recvfrom(nl, buf, sizeof(buf));
	}

	mnl_socket_close(nl);

	return 0;
}

```
```text
gcc rtnl-route-get.c -lmnl
```




