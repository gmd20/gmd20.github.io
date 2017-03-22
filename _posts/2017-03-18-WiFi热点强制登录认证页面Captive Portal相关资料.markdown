什么是Captive Portal
===================
大家肯定都连过公共场所的wifi热点，比如麦当劳等地方的。他们的wifi往往一连上去就会弹出一个要求登录或者微信关注之类的页面，只有在这个页面完成操作了才能正常访问网络的。之前看到这个很神奇，为什么一连wifi，手机就会自动打开这个网页的，就知道android系统应该是提供了一些接口的。最近接触到这个，查了一下才知道这个东西叫做“captive portal”，就是专门用来给后端的网关提供鉴权计费之类的服务的。很多公共场合的wifi热点应该都用了这么一个技术，比如酒店，商场，银行等等。


Captive Portal的原理
===================
Captive Portal这个东西应该是随着wifi的流行，很多公共场所都有这种需求，才慢慢形成的一个概念。但好像各大手机厂商和操作系统都还没有形成一个统一的标准（好像有个Hotspot 2.0 IEEE 802.11u 的标准，好像新的ios 和三星手机和windows 10都支持。微软的文档还提到一个 Wireless Internet Service Provider roaming (WISPr) 但好像被抛弃了。）。大概的实现都是类似这样子的，系统/浏览器 检查到网络不正常了或者在wifi刚连接的时候，他就假设这个后面的网关是要求登录验证的，假设这个网络是不正常的，然后运行一个“网络探测/网络诊断”的逻辑，简单的说就是去尝试访问一个指定的http服务器的指定页面，如果这页面返回的结果(网页内容或者http status code)跟自己预期的不一样，就认为这个返回的页面是中间的网关人为纂改了得http内容，认为这个返回的页面网关提供的“强制登陆页面captive portal”，就直接显示这个内容让用户进行登录认证操作。
但具体到哥哥操作系统略有差异：

* Windows 7/ Windows 10 微软家族
  windows 10 据说会访问 www.msftconnecttest.com/connecttest.txt  和 ipv6.msftconnecttest.com/connecttest.txt 这几个地址，然后根据
  这个返回的结果来判断是否是认证页面。微软的官网文档还算比较详细，给了很多指导性的建议，比如怎么实现网关的captive portal功能，看上去
  还能返回一个特定格式的xml，让手机直接显示比较好看的页面或者安装APP等，系统可以提供“Handling the hotspot authentication event”事件通知
  让第三方APP来处理这个验证请求？可能一些“wifi万能密码“的之类的应用会监控处理这种事件？
  可以自己查看文章后面提供的微软msdn里面captive portal说明文档。
  windows 10 里面可以通过修改注册表的HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\NlaSvc\Parameters\Internet 项下面的
  参数来配置是否“主动探测网络”和“具体的网址，dns”和探测时间间隔等。
 
* 苹果iOS
  iOS也会在网络探测里面访问一个固定的网页地址（http://captive.apple.com/hotspotdetect.html），用以判断是否captive portal页面。
  但好像网上说不同ios版本这个网址稍微有所差异，有人说http请求里面的头部会有 User Agent 'CaptiveNetworkSupport' 标志。
  好像ios也有类似上面说的windows phone的 “网络认证”事件通知给第三方APP来处理认证的接口，让app选择哪个热点之类的。
  但ios的资料没仔细看，没找到官方的文档。
  
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
  网上有人写
```
One section of www.google.com/chrome/browser/privacy/whitepaper.html reads:

In the event that Chrome detects SSL connection timeouts, certificate errors, or other network issues that might be caused by a captive portal (a hotel's WiFi network, for instance), Chrome will make a cookieless request to http:// www.gstatic.com/generate_204 and check the response code. If that request is redirected, Chrome will open the redirect target in a new tab on the assumption that it's a login page. Requests to the captive portal detection page are not logged.
It's actually sending a request to http://connectivitycheck.android.com/generate_204. 

```

实现Captive Portal相关的技术
===========================
* ICMP redirect
  wiki提到这个， 但这个好像比较难用？ 通知client把消息转发给另外的路由路径发送，好像比较难以控制。

* Redirect by DNS  
  DNS劫持， 这个好像比较简单，在浏览器解析域名的时候，网关拦截dns请求，直接给你返回一个TTL = 0的假的http服务器ip地址给你。
  就直接跳转到登录页面服务器上了。 TTL = 0 对系统dns缓存影响很小，下次系统再重新发送dns请求就可以得到正确的地址了。
  估计路由里面管理页面   http://tplogin.cn  这样的是通过这个技术实现的。
  ```
  C:\nslookup tplogin.cn
服务器:  ns5.gd.cnmobile.net
Address:  211.136.192.6

非权威应答:
名称:    tplogin.cn
Addresses:  192.168.1.1
          192.168.1.1
  ```
 
* HTTP劫持
  直接在网关修改http的tcp网络包，通过一个NAT重定向到 认证服务器，或者给客户端返回 http status code 302 通知浏览器跳转网页。
  其他复杂的修改还有直接修改http 返回的内容，插入广告脚本等等。
  我猜测我这里移动光纤宽带，老弹出广告页面，应该是是用了这个技术了，返回一个302可以让你重定向到他的网站去。
  
* HTTPS 劫持/中间人欺骗
  劫持HTTPS好像不太可行，因为中间网关不能拿到通讯内容了，无法纂改。本来人家HTTPS就是为了防止纂改的。以前的sslstrip本质上还是修改http，
  出来这个之后，浏览器和系统搞了个http-strict-transport-security 的增强http头选项，让浏览器可以记住上次访问时https，这样用户下次再
  输入不带http或者https的地址时，浏览器默认也是用https发起了，sslstrip就不能工作了。  如果简单的dns劫持 https的地址或者从定向到假的
  服务器地址上去。这个伪造的服务器不能为所有的https网站都返回合适的证书。 浏览器检查服务器返回的证书里面的subject name 可以发现跟域名
  不一致就知道是假的了。而且这种伪造的服务器证书是自己颁发的，也是不合格的。浏览器可以识别出这个伪造的https服务器来，给用户显示警告消息
  提醒用户。 至于通过系统漏洞提权绕过浏览器的”警告消息“应该也是没有可行性的，终端太多类型。
  
  
captive portal开源软件
=====================
[wifidog](http://dev.wifidog.org/)
NoCat
nodogsplash
好像商业的也很多，国内都很多提供这个功能的，可以搜索一下。看到的有结合短信做验证的，也有结合微信做认证的等等。


查看的网页，可能不限于这些
========================
[https://blogs.windows.com/msedgedev/2015/06/09/http-strict-transport-security-comes-to-internet-explorer-11-on-windows-8-1-and-windows-7/#pocTdcHEsSV4ih1O.97] (https://blogs.windows.com/msedgedev/2015/06/09/http-strict-transport-security-comes-to-internet-explorer-11-on-windows-8-1-and-windows-7/#pocTdcHEsSV4ih1O.97)
[https://msdn.microsoft.com/windows/hardware/drivers/mobilebroadband/captive-portals](https://msdn.microsoft.com/windows/hardware/drivers/mobilebroadband/captive-portals)
[http://stackoverflow.com/questions/18891706/ios7-and-captive-portals-changes-to-apple-request-url](http://stackoverflow.com/questions/18891706/ios7-and-captive-portals-changes-to-apple-request-url)
[http://www.chromium.org/chromium-os/chromiumos-design-docs/network-portal-detection](http://www.chromium.org/chromium-os/chromiumos-design-docs/network-portal-detection)
[http://blog.csdn.net/innost/article/details/20863901](http://blog.csdn.net/innost/article/details/20863901)
[https://mp.weixin.qq.com/wiki/10/0ef643c7147fdf689e0a780d8c08ab96.html] (https://mp.weixin.qq.com/wiki/10/0ef643c7147fdf689e0a780d8c08ab96.html)
[https://developer.android.google.cn/reference/android/net/CaptivePortal.html](https://developer.android.google.cn/reference/android/net/CaptivePortal.html)
[http://serverfault.com/questions/811005/captive-portal-issue-with-chrome-on-android](http://serverfault.com/questions/811005/captive-portal-issue-with-chrome-on-android)
[https://support.apple.com/en-us/HT204497] (https://support.apple.com/en-us/HT204497)
[http://blog.erratasec.com/2010/09/apples-secret-wispr-request.html#.VxkmrJMrIy5](http://blog.erratasec.com/2010/09/apples-secret-wispr-request.html#.VxkmrJMrIy5)
[https://www.reddit.com/r/Windows10/comments/4p6r9r/windows_10_captive_portal_detection/](https://www.reddit.com/r/Windows10/comments/4p6r9r/windows_10_captive_portal_detection/)
[http://blog.superuser.com/2011/05/16/windows-7-network-awareness/](http://blog.superuser.com/2011/05/16/windows-7-network-awareness/)
[http://community.arubanetworks.com/t5/Technology-Blog/RFC-7710-Captive-Portal-Identification-Using-DHCP-or-Router/ba-p/255737](http://community.arubanetworks.com/t5/Technology-Blog/RFC-7710-Captive-Portal-Identification-Using-DHCP-or-Router/ba-p/255737)


  
