由pimd这样的后台应用程序和内核交互，控制内核里面的组播路由转发表吧。


https://flylib.com/books/en/3.475.1.88/1/

https://www.man7.org/linux/man-pages/man8/ip-mroute.8.html

https://github.com/troglobit/pimd/tree/master

https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/mroute.h

pimd的介绍
https://www.cnblogs.com/newjiang/p/13706485.html
https://blog.csdn.net/wuheshi/article/details/103049441

组播转发开关
echo 1 > /proc/sys/net/ipv4/conf/all/mc_forwarding
