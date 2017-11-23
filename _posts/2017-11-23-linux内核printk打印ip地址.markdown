https://www.kernel.org/doc/Documentation/printk-formats.txt
http://elixir.free-electrons.com/linux/v4.14.1/source/Documentation/printk-formats.txt
```text

打印时，还可以选择字节大小端顺序的比较方便

IPv4 addresses
==============

::

	%pI4	1.2.3.4
	%pi4	001.002.003.004
	%p[Ii]4[hnbl]

For printing IPv4 dot-separated decimal addresses. The ``I4`` and ``i4``
specifiers result in a printed address with (``i4``) or without (``I4``)
leading zeros.

The additional ``h``, ``n``, ``b``, and ``l`` specifiers are used to specify
host, network, big or little endian order addresses respectively. Where
no specifier is provided the default network/big endian order is used.

Passed by reference.

IPv6 addresses
==============

::

	%pI6	0001:0002:0003:0004:0005:0006:0007:0008
	%pi6	00010002000300040005000600070008
	%pI6c	1:2:3:4:5:6:7:8

For printing IPv6 network-order 16-bit hex addresses. The ``I6`` and ``i6``
specifiers result in a printed address with (``I6``) or without (``i6``)
colon-separators. Leading zeros are always used.

The additional ``c`` specifier can be used with the ``I`` specifier to
print a compressed IPv6 address as described by
http://tools.ietf.org/html/rfc5952

Passed by reference.

IPv4/IPv6 addresses (generic, with port, flowinfo, scope)
=========================================================

::

	%pIS	1.2.3.4		or 0001:0002:0003:0004:0005:0006:0007:0008
	%piS	001.002.003.004	or 00010002000300040005000600070008
	%pISc	1.2.3.4		or 1:2:3:4:5:6:7:8
	%pISpc	1.2.3.4:12345	or [1:2:3:4:5:6:7:8]:12345
	%p[Ii]S[pfschnbl]

For printing an IP address without the need to distinguish whether it``s
of type AF_INET or AF_INET6, a pointer to a valid ``struct sockaddr``,
specified through ``IS`` or ``iS``, can be passed to this format specifier.

The additional ``p``, ``f``, and ``s`` specifiers are used to specify port
(IPv4, IPv6), flowinfo (IPv6) and scope (IPv6). Ports have a ``:`` prefix,
flowinfo a ``/`` and scope a ``%``, each followed by the actual value.

In case of an IPv6 address the compressed IPv6 address as described by
http://tools.ietf.org/html/rfc5952 is being used if the additional
specifier ``c`` is given. The IPv6 address is surrounded by ``[``, ``]`` in
case of additional specifiers ``p``, ``f`` or ``s`` as suggested by
https://tools.ietf.org/html/draft-ietf-6man-text-addr-representation-07

In case of IPv4 addresses, the additional ``h``, ``n``, ``b``, and ``l``
specifiers can be used as well and are ignored in case of an IPv6
address.

Passed by reference.

Further examples::

	%pISfc		1.2.3.4		or [1:2:3:4:5:6:7:8]/123456789
	%pISsc		1.2.3.4		or [1:2:3:4:5:6:7:8]%1234567890
	%pISpfc		1.2.3.4:12345	or [1:2:3:4:5:6:7:8]:12345/123456789

```
