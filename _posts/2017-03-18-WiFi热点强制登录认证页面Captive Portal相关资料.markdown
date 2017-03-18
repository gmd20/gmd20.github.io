什么是Captive Portal
===================
大家肯定都连过公共场所的wifi热点，比如麦当劳等地方的。他们的wifi往往一连上去就会弹出一个要求登录或者微信关注之类的页面，只有在这个页面完成操作了才能正常访问网络的。之前看到这个很神奇，为什么一连wifi，手机就会自动打开这个网页的，就知道android系统应该是提供了一些接口的。最近接触到这个，查了一下才知道这个东西叫做“captive portal”，就是专门用来给后端的网关提供鉴权计费之类的服务的。很多公共场合的wifi热点应该都用了这么一个技术，比如酒店，商场，银行等等。


Captive Portal的原理
===================
Captive Portal这个东西应该是随着wifi的流行，很多公共场所都有这种需求，才慢慢形成的一个概念。但好像各大手机厂商和操作系统都还没有形成一个统一的标准（好像有个Hotspot 2.0 IEEE 802.11u 的标准，但不知道各个无线路由器和手机支持的怎么样。）。大概的实现都是类似这样子的，系统/浏览器 检查到网络不正常了或者在wifi刚连接的时候，他就假设这个后面的网关是要求登录验证的，假设这个网络是不正常的，然后运行一个“网络探测/网络诊断”的逻辑，简单的说就是去尝试访问一个指定的http服务器的指定页面，如果这页面返回的结果(网页内容或者http status code)跟自己预期的不一样，就认为这个返回的页面是中间的网关人为纂改了得http内容，认为这个返回的页面网关提供的“强制登陆页面captive portal”，就直接显示这个内容让用户进行登录认证操作。
但具体到哥哥操作系统略有差异：

* Windows 7/ Windows 10 微软家族
  windows 10 据说会访问 www.msftconnecttest.com/connecttest.txt  和 ipv6.msftconnecttest.com/connecttest.txt 这几个地址，然后根据
  这个返回的结果来判断是否是认证页面。微软的官网文档还算比较详细，给了很多指导性的建议，比如怎么实现网关的captive portal功能，看上去
  还能返回一个特定格式的xml，让手机直接显示比较好看的页面或者安装APP等，系统可以提供“Handling the hotspot authentication event”事件通知
  让第三方APP来处理这个验证请求？可能一些“wifi万能密码“的之类的应用会监控处理这种事件？
  可以自己查看文章后面提供的微软msdn里面captive portal说明文档。
  
* 苹果iOS
  iOS也会在网络探测里面访问一个固定的网页地址，用以判断是否captive portal页面。但好像网上说不同ios版本这个网址稍微有所差异，有人说http请求
  里面的头部会有 User Agent 'CaptiveNetworkSupport' 标志。好像ios也有类似上面说的windows phone的 “网络认证”事件通知给第三方APP来处理认证
  的接口。但ios的资料没仔细看，没找到官方的文档。
  
* Android
  Android会访问 http://connectivitycheck.android.com/generate_204  这样的页面来判断，看看返回的http status code是不是204.
  android的这个页面地址可以修改(不同版本好像名字稍微不太一样):
  ```
  adb shell "settings put global captive_portal_http_url http://captive.v2ex.co/generate_204"
  adb shell "settings put global captive_portal_https_url https://captive.v2ex.co/generate_204"
  ```
  android的应该文档比较详细一些，可以查看对应的系统怎么处理这个”captive portal“逻辑的代码，网上有人分析这个开源代码。
  也有应用专门修改这个页面地址的，说是原生的地址在墙外导致android网络探测这段逻辑工作不正常。真要研究应该可以结合源码
  分析就清楚了。
  
  
* Chrome/Chromium浏览器


