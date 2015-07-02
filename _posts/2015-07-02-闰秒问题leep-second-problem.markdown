听说我们买的网络设备（服务器）由于6月30好的闰秒问题，部署在不同地区的全部这种服务器全部挂了。
说是cpu利用率突然变得很高。

看上去是旧版本的linux内核更新闰秒的bug，导致futex这些定时器不断超时重试。

这里有一些leap second可能导致cpu load 很高的原因的解释

http://www.pythian.com/blog/handling-the-leap-second-linux/
http://developerblog.redhat.com/2015/06/01/five-different-ways-handle-leap-seconds-ntp/
http://www.madore.org/~david/computers/unix-leap-seconds.html
