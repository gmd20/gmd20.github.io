iPhone里面有两种探测方式：
========================

方法一：
-------
猜测这个对应首次连接一个新的SSID，经过了文档提到的hotspothelper的“Evaluate”流程，
对应文档的“Sequence When Network Is Captive and UI Is Required” 流程。
不管有人说apple会使用了随机使用多个域名来探测。

1. 第一个HTTP请求
```text
   captive.apole.com
         GET  /hotspot-detect.html HTTP/1.0
         User-Agent: CaptiveNetworkSupport-346.50.1 wispr\r\n
```

2. 第二个HTTP请求
```text
   www.apple.com
         GET / HTTP/1.1
         User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 10_3_1 like Mac OS X) AppleWebKit/603.1.30 (KHTML, like Gecko) Mobile/14E304\r\n
```

3.  第二步页面加载完之后，40毫秒后发送探测
```text
    www.apple.com
         GET /library/test/success.htm  HTTP/1.0
         User-Agent: CaptiveNetworkSupport-346.50.1 wispr
```
4.  用户提交数据后，连发两个第3步一样的探测包。


方法二：
-------
猜测这个对应已经连过了的SSID，缓存直接找到best_helper 没有“Evaluate”阶段的流程，
对应文档的“Sequence When Network Is Captive (Cached)”，直接从Maintaining state
开始认证流程。


 1. 第一个HTTP请求
```text
     captive.apole.com
         GET  /hotspot-detect.html HTTP/1.0
         User-Agent: CaptiveNetworkSupport-346.50.1 wispr\r\n
```

 2. 第二个HTTP请求
```text
    100毫秒秒后
     captive.apole.com
         GET  /hotspot-detect.html HTTP/1.1
         User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 10_3_1 like Mac OS X) AppleWebKit/603.1.30 (KHTML, like Gecko) Mobile/14E304\r\n
```

 3.  等第2步的页面加载完后，过了200毫秒，又马上发送探测
```text
     captive.apole.com
         GET  /hotspot-detect.html HTTP/1.0
         User-Agent: CaptiveNetworkSupport-346.50.1 wispr\r\n
```

 4.  第2步用户操作提交数据后，过了400毫秒，
     马上连发2个和第3步一样的探测。




总结一下iPhone的认证页面跳转流程
================================
```text
       +---------------------------------+
       | “扫描附近wifi热点” scan 阶段    |       这个阶段iOS遍历所有的 helper，各个助手模块自己给wifi加注释（
       +-------------------+-------------+       界面可以看到的wifi热点下面的描述吧）和提供wifi连接密码。
                           |                     各个助手模块应该也可以强制设置某个wifi热点不需要认证，直接自动登录等。
                           |
                           |
          缓存里面是否有这个wifi热点的记录
                           |
                           |
                           +------------------------------->+
                           |                                |
                       否  |                                |  是
                           |                                |
                           v                                v
     +-------------------------+                     +-------------------------------+
     |  Evaluate阶段           |                     |    Maintain阶段               |
     |  CNA 发送 HTTP/1.0探测  |                     |                               |  <----+
     |  参考下文的说明         |                     | 每隔300秒CNA发送HTTP/1.0 探测 |       |
     +----------+--------------+                     +-------------------------------+       |
                |                                       |                                    |
                |                                       |                                    |
                |                                       |                                    |
                |                                 需要重复认证                               |
                |                                       |                                    |
       是不是需要portal认证的网络                       |                                    |
                |                                       |                                    |
                |                                       |                                    |
                |                                       |                                    |
                |                                       v                                    |
                |         是              +-------------------+                              |
                +-----------------------> | Authenticate      |                              |
                |                         +-------------------+------------------+           |
                |                         | PresentUI阶段 弹出认证窗口           |           |
                |                         |                                      |           |
                |                         | CNA发送 HTTP/1.1 请求加载portal页面  |           |
                |                         | 之后每次portal页面加载完成都会触发   |           |
                |                         | HTTP/1.0探测 。如果HTTP/1.1连接超时  |           |
                |                         | 会显示空白页，最后会报告服务器超时   |           |
                |                         +---------------------+----------------+           |
                |                                               |                            |
             否 |             +---------------------------------+                            |
                |             |                                                              ^
                |      HTTP/1.0 探测显示网络正常                                             |
                |             |                                                              |
                v             v                                                              |
   +------------+-------------+---------+                                                    |
   |  认证完成，wifi “完成”按钮可用     |  ------------------------------------------------>-+
   +------------------------------------+
```


其他观察结果和结论：
===================
1.  iPhone 不会对
```text
    captive.apple.com或者www.apple.com
         GET  /hotspot-detect.html HTTP/1.0
         User-Agent: CaptiveNetworkSupport-346.50.1 wispr\r\n
```


    这个HTTP/1.0请求返回的的http response 进行任何处理，如果这个respone里面有各种跳转， iPhone也不会执行跳转。
    页面跳转只会在 HTTP/1.1那个的结果里面进行。

2.  跳转出来的Portal页面每次刷新，都会触发HTTP/1.0的探测包。 如果这个请求到了苹果的服务器
    回复了Sucess页面的话（参考下面），iPhone就认为wifi网络正常，UI上面的 “完成”按钮就变为可用。
    它这个探测发送的很快， 所以必须确保网关放行规则起作用后，页面还能刷新一次或者放行规则起
    作用后页面再返回，触发这个探测来点亮图标。

3.  利用http status code 302/303的跳转portal页面的方式，相比较javascript的跳转方式，
    前者可以让iPhone少发一次HTTP/1.0探测包。

4.  安装“wifi万能钥匙”的情况下，wifi万能钥匙和iphone自身都会发送探测包。
    “wifi万能钥匙” 不但用iPhone还会用android的方式的HTTP来探测网络。


wifi万能钥匙发起的iphone探测
----------------------------
```text
    GET / HTTP/1.1\r\n
    Host: captive.apple.com\r\n
    User-Agent: Zeus/95 CFNetwork/811.4.18 Darwin/16.5.0\r\n
    Accept-Language: zh-cn\r\n
    Connection: keep-alive\r\n
    [Full request URI: http://captive.apple.com/]
```

Apple服务器给它的回应
-----------------------------
```text
    HTTP/1.1 304 Not Modified\r\n
    Cache-Control: max-age=300\r\n
    Connection: keep-alive\r\n
    Via: http/1.1 uslax1-edge-bx-005.ts.apple.com (ApacheTrafficServer/7.0.0)\r\n
    Server: ATS/7.0.0\r\n
```


wifi万能钥匙发送的探测
----------------------
```text
    GET /generate_204 HTTP/1.1\r\n
    Host: c.51y5.net\r\n
    Accept: */*\r\n
    Accept-Language: zh-cn\r\n
    Connection: keep-alive\r\n
    Accept-Encoding: gzip, deflate\r\n
    User-Agent: Zeus/95 CFNetwork/811.4.18 Darwin/16.5.0\r\n
    \r\n
```

wifi万能钥匙自己服务器发送的andoird探测包回应
---------------------------------------------
```text
    HTTP/1.1 204 No Content\r\n
        [Expert Info (Chat/Sequence): HTTP/1.1 204 No Content\r\n]
        Request Version: HTTP/1.1
        Status Code: 204
        Response Phrase: No Content
    Server: nginx\r\n
    Connection: keep-alive\r\n
```



正常iphone 发送的探测包 和回应
------------------------------
```text
    GET /hotspot-detect.html HTTP/1.0\r\n
    Host: captive.apple.com\r\n
    Connection: close\r\n
    User-Agent: CaptiveNetworkSupport-346.50.1 wispr\r\n
```

正常iPhone 服务器探测包的回应
------------------------------
```text
    HTTP/1.0 200 OK\r\n
    Cache-Control: max-age=300\r\n
    Accept-Ranges: bytes\r\n
    Content-Type: text/html\r\n
    Server: ATS/7.0.0\r\n
    CDNUUID: 8ec1d738-abf5-4482-ae50-d04bbd52e601-1473127713\r\n
    Via: https/1.1 jptyo5-edge-lx-010.ts.apple.com (ApacheTrafficServer/7.0.0), http/1.1 jptyo5-edge-bx-004.ts.apple.com (ApacheTrafficServer/7.0.0)\r\n
    X-Cache: hit-fresh, hit-fresh\r\n
    Etag: "41ba060eb1c0898e0a4a0cca36a8ca91"\r\n
    Age: 116\r\n
    \r\n
    <HTML><HEAD><TITLE>Success</TITLE></HEAD><BODY>Success</BODY></HTML>\n
```


可以知道wifi万能钥匙的iPhone方式探测包url有点问题的。


5.  如果某次页面弹出有问题，“完成” 按钮没有亮， 客人又点 “取消”，“继续使用网络”，那么
    “auto-join”按钮会被禁用。 这个应该对应文档里面的 kNEHotspotHelperCommandTypePresentUI状态
    返回“kNEHotspotHelperResultFailure (or any other result) A fatal error occurred; auto-join disabled.”
    然后客人如果选择继续“使用网络”后。 在断开，重连到wifi热点来，因为auto-join被取消了，认证页面
    也是不会再自动弹出来的。

6. Evaluate阶段导致的45超时问题
   发现iPhone首次连新的ssid的热点时，弹出页面都是在45秒左右。（ios 10.3.1版本 安装有“wifi万能驱动”）
   这个原因应该是首次连接时， iPhone默认执行“Evaluate”流程，给所有的系统所有的hotspothelper模块都发送
   “Evaluate” 命令，系统部本身自带一个Captive Network Assitant（CNA） helper，安装了“Wifi万能驱动”
   “QQ浏览器wifi助手” 之后的APP之后，他们也都安装所有的Helper模块。
   iOS系统发所有的模块下发命令后，要求各个模块回复“网络是否可用的评估值”。一旦有任何一个helper模块回复 “high”
   表示这个模块很自信自己能够处理接下来的“登录认证”，那么其他没有返回结果的helper模块切换到后台，他们的结果
   也会被忽略。如果所有的helper模块都没有任何回复，那么iOS直接认为网络可用，状态直接变成Authenticated“认证完成”。

   而出现问题的原因，打开是各个wifi助手app没有实现这个Evaluate命令，或者由于某种原因一直不回复Evaluate命令。
   系统默认的CNA helper的Evaluate被调用了使用 HTTP1.0 探测 captive.apple.com，但CNA helper了返回的结果应该是“Low”
  （不是很确定自己能不能处理后面的认证）。这样iOS还没看到一个“high”的结果，又有其他helper没有回复，就会一直等待
   到45秒的超时结束。过了45秒之后才能勉强的把这个“low”结果的CNA作为best helper，在继续下面的 “Authenticating”。
   这个其实是各个APP没有响应这个Evaluate命令造成的，如果助手APP快速返回“low”，iOS拿到所有helper的结果不会等待
   这个45秒了。虽然这样处理iOS看到多个low的回复，只能任意选一个进行下面的“认证”了，认证的时候各个模块可以
   再直接返回失败的让iOS把自己从helper列表删除，iOS会把不支持的helper 排除在外，然后再重做Evalute。
   听说有厂商已经给这些“助手类hotspothelper”的开发者报告问题，大多的APP在最新版里面已经修复这个问题。
   其实如果系统默认的CNA helper返回的是“high”也能避免这个超时等待，但对应该新的网络SSID，它确实不能保证后面能
   不能认证成功返回low也是合理的。 如果自己实现APP helper模块，可以通过ssid和网络探测等方式知道是自己能处理的
   wifi热点，直接返回high也是能够避免这个45秒的超市等待的。

   但如果使用“CNA”成功做过一次portal认证之后，是没有这个45秒超时等待问题了。因为iOS有个cache缓存机制，第二次
   连接的时候他通过cache知道这个“CNA” helper模块能够处理这个网络的认证，那么他就跳过“Evaluate”阶段，直接让
   CNA helper开始“Maintaining”处理。所以第二次以后就没有这个“Evaluate”的45秒超时等待了。

   *根据apple开发者论坛的说法在iOS的任务管理器里面把“wifi助手”进程给结束掉，iOS就也不会调用这些APP的hotspothelper*
   这样没有其他app hotspothelper的干扰，应该也可以避免这个首次连接的超时问题。

7. 这个HotspotHelper接口按apple的说法是专门给用了开发“captive portal”的app用的，专门用来辅助wifi热点认证的。
   hotspothelper模块在整个认证过程中出现错误，可能导致iOS把这个模块从这个wifi网络的“有效列表”中删除，不再参与后面的
   重试。可能 wifi热点还会被 “断开” ，或者wifi热点的“auto-join”按钮被禁用等。这都可能影响到最终的认证行为。具体可以
   参考Apple的状态说明。

8. 其他导致页面显示慢的问题，估计主要是网络原因导致 HTTP/1.0和 HTTP/1.1两个连接创建失败或者超时导致的
   如果到了“PresentUI阶段”，但HTTP1.1的请求portal页面的请求失败了，那么就显示的是空白页面了。其他的影响
   因素还有dns解析的速度，网关或者防火墙是否有连接数量限制导致的丢包等等。
   可以pc连接上网络，然后暂时不认证portal页面，然后用一些http测试工具测试一下portal服务器连接的性能，抓包
   看看，比如用这个工具https://github.com/rakyll/hey
   ./hey -more=1  -n 256  -c 4 -q 4  http://captive.apple.com/hotspot-detect.htm

   另外需要避免portal页面的http服务打开了TCP的tcp_tw_recycle选项，然后用户全部通过NAT网络连接这个服务器。
   这样tcp的timestamp选项和tcp_tw_recycle一起使用时， NAT网络用户的创建连接会很大概率失败。
   原因这两篇文章有介绍：
   一个NAT问题引起的思考
   http://perthcharles.github.io/2015/08/27/timestamp-NAT/
   Coping with the TCP TIME-WAIT state on busy Linux servers
   https://vincent.bernat.im/en/blog/2014-tcp-time-wait-state-linux

9. 按照苹果文档的说法，wifi热点还没有认证通过的时候（“非Authenticated”），默认
   路由是不会是wifi设备的，自己的helper里面需要从wifi上面发送探测网络包，是需要
   明确指定了出去的网络设备才行的。但在实际测试发现，不知道是不是一些助手helper
   模块的干扰，有的时候是会看到一些 “支付宝”或者“微信”等的443端口的https还有8080
   或者80的端口出来。有可能这是wifi被错误的设置了 authenticated 状态，又要等到
   后面CNA探测到问题再开始显示portal页面。但在这之前已经有很多后台页面尝试在wifi
   设备上发送请求了，有的比如qq的浏览器加载portal页面，还会自己尝试加载重定向后
   的页面等等。


测试方法
========
笔记本windows 10系统 + “移动热点” + 手机无线连笔记本 + dns劫持+ wireshark抓包

配置测试环境
右键 开始菜单 -> “设置“ -> "网络设置" -> “移动热点”
在网络设备里面找到移动热点对应的  网络设备，修改 dns服务器地址为本地虚假dns服务器的ip。
这个ip 不要用和 改设备一样的，使用本机的其他ip比如承载网卡的ip。
保证 手机端获取的dns转到自己的dns服务器就可以了。
用于测试golang dns server 代码。
```go
package main

import (
	"flag"
	"fmt"
	"github.com/miekg/dns"
	"html/template"
	"log"
	"math/rand"
	"net"
	"net/http"
	"os"
	"regexp"
	"strings"
	"sync"
	"time"
)

const (
	DEFAULT_PORT    = 53
	DEFAULT_TIMEOUT = 20 * time.Second
	DEFAULT_PROTO   = "udp"
)

var (
	LOG       *log.Logger
	FAKE_IP   net.IP
	PROTO     string
	TIMEOUT   time.Duration
	DEBUG     bool
	TCP_REGEX *regexp.Regexp

	DEFAULT_SERVERS = []string{
		// "8.8.8.8",      // google
		// "8.8.4.4",      // google
		// "208.67.222.222", // OpenDNS
		// "208.67.220.220", // OpenDNS
		"114.114.114.114", // 114dns
		"180.76.76.76",    // baidu
		"223.5.5.5",       // ali
		"223.6.6.6",       // ali
		"119.29.29.29",    // tencent dnspod
	}

	LOGIN_USER = make([]int64, 0, 8)
	USER_MUTEX = &sync.RWMutex{}

	REDIRECT_MODE int
)

func ipv4ToInt64(ip net.IP) int64 {
	if DEBUG {
		fmt.Println("ip to int64: ", ip)
	}
	ip4 := ip.To4()
	if len(ip4) == net.IPv4len {
		n := (int64(ip4[0]) << 24) | (int64(ip4[1]) << 16) | (int64(ip4[2]) << 8) | int64(ip[3])
		return n
	}
	return 0
}
func userExist(ip int64) bool {
	USER_MUTEX.RLock()
	found := false
	for i := 0; i < len(LOGIN_USER); i++ {
		if LOGIN_USER[i] == ip {
			if DEBUG {
				fmt.Println("User Exist: ", ip)
			}
			found = true
			break
		}
	}
	USER_MUTEX.RUnlock()
	return found
}
func addUser(ip int64) {
	if !userExist(ip) {
		if DEBUG {
			fmt.Println("Add User:", ip)
		}
		USER_MUTEX.Lock()
		LOGIN_USER = append(LOGIN_USER, ip)
		USER_MUTEX.Unlock()
	}
}

func init() {
	LOG = log.New(os.Stderr, "[DNS PROXY] ", log.LstdFlags)

	var proto, pattern, fakeIP string
	var timeout int

	flag.BoolVar(&DEBUG, "debug", false, "调试开关")
	flag.StringVar(&proto, "protocol", "udp", "udp|tcp 用udp还是tcp连接远程dns服务器")
	flag.IntVar(&timeout, "timeout", 0, "timeout 等待远程dns服务器的超时时间, 单位秒")
	flag.StringVar(&pattern, "regex", ".*", "需要过滤域名的正则表达式,默认所有域名都返回假ip")
	flag.StringVar(&fakeIP, "fake_ip", "", "dns查询返回的假ip地址")
	flag.IntVar(&REDIRECT_MODE, "redirect_mode", 0, "网页重定向的方式 0到4数字")
	flag.Parse()

	switch proto {
	case "tcp":
		PROTO = "tcp"
	case "udp":
		PROTO = "udp"
	default:
		PROTO = DEFAULT_PROTO
	}

	if timeout > 5 {
		TIMEOUT = time.Duration(timeout) * time.Second
	} else {
		TIMEOUT = DEFAULT_TIMEOUT
	}

	if len(pattern) > 0 {
		if re, err := regexp.Compile(pattern); err != nil {
			LOG.Fatalf("Compiling pattern [%s] was %s", pattern, err)
		} else {
			TCP_REGEX = re
		}
	}
	FAKE_IP = net.ParseIP(fakeIP)

	LOG.Printf("Use %s protocol to connect to the remote DNS server", PROTO)
	LOG.Printf("Timeout duration: %s", TIMEOUT)
	if TCP_REGEX != nil {
		LOG.Printf("Compiling tcp regex pattern [%s]", TCP_REGEX)
	}
	LOG.Printf("Use fake IP %s", fakeIP)
	LOG.Println("Redirect mode: ", REDIRECT_MODE)
}

type Proxy struct {
	Servers []string
}

func (p Proxy) ServeDNS(w dns.ResponseWriter, req *dns.Msg) {
	if TCP_REGEX != nil && FAKE_IP != nil {
		q := req.Question[0]

		isknownUser := false
		ip, _, err := net.SplitHostPort(w.RemoteAddr().String())
		if DEBUG {
			fmt.Println("dns client ip: ", ip)
		}
		if err == nil {
			userIP := net.ParseIP(ip)
			if userIP != nil {
				isknownUser = userExist(ipv4ToInt64(userIP))
			}
		}
		if TCP_REGEX.MatchString(q.Name) && !isknownUser {
			if DEBUG {
				LOG.Printf("DEBUG return a fake ip: %s => %s", q.Name, FAKE_IP)
			}
			m := new(dns.Msg)
			m.SetReply(req)
			m.Authoritative = true
			m.RecursionAvailable = true
			m.Answer = make([]dns.RR, 0, 1)
			m.Answer = append(m.Answer,
				&dns.A{
					Hdr: dns.RR_Header{
						Name:   q.Name,
						Rrtype: dns.TypeA,
						Class:  dns.ClassINET,
						Ttl:    0,
					},
					A: FAKE_IP.To4(),
				})
			w.WriteMsg(m)
			return
		}
	}

	c := new(dns.Client)

	c.Net = PROTO
	// if TCP_REGEX != nil {
	//	for _, q := range r.Question {
	//		if TCP_REGEX.MatchString(q.Name) {
	//			LOG.Printf("Tcp proto regex match: %s", q.Name)
	//			c.Net = "tcp"
	//			break
	//		}
	//	}
	// }

	c.ReadTimeout = TIMEOUT
	c.WriteTimeout = TIMEOUT

	if rs, _, err := c.Exchange(req, p.Server()); err == nil {
		if DEBUG {
			LOG.Printf("DEBUG %s\n%v", w.RemoteAddr(), rs)
		}
		w.WriteMsg(rs)
	} else {
		m := new(dns.Msg)
		m.SetRcode(req, dns.RcodeServerFailure)
		w.WriteMsg(m)
		LOG.Printf("%s %s", w.RemoteAddr(), err)
	}
}

func (p Proxy) Server() string {
	sl := len(p.Servers)
	if sl > 0 {
		i := rand.Intn(sl)
		s := p.Servers[i]
		if strings.Index(s, ":") == -1 {
			s = fmt.Sprintf("%s:%d", s, DEFAULT_PORT)
		}
		return s
	}
	return "8.8.8.8:53"
}

// ----------------------------------------------------------------------------

type PortalServer struct{}

func (h *PortalServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	var urlPath = r.URL.Path
	fmt.Println("query url:", urlPath)
	if urlPath == "/login.go" {
		if len(r.FormValue("UserName")) != 0 {
			if len(r.FormValue("WISPrVersion")) != 0 ||
				len(r.FormValue("FNAME")) != 0 ||
				len(r.FormValue("OriginatingServer")) != 0 {
				fmt.Println("用户使用wispr方式登录")
			}

			fmt.Fprint(w, "<HTML><HEAD><TITLE>Success</TITLE></HEAD><BODY>"+r.FormValue("id")+" 登录成功"+"</BODY> </HTML>")
			ip, _, err := net.SplitHostPort(r.RemoteAddr)
			if err != nil {
				fmt.Println("Failed to SplitHostPort:", r.RemoteAddr)
				return
			}
			userIP := net.ParseIP(ip)
			if userIP == nil {
				fmt.Println("Failed to ParseIP:", ip)
				return
			}
			addUser(ipv4ToInt64(userIP))
			// 页面返回后，iPhone很快就下发探测包。我们慢点回复确保它探测的时候AddUser
			// 规则已经其作用了。
			time.Sleep(500 * time.Millisecond)
			if DEBUG {
				fmt.Println("user has submit id to login.go:", ip)
			}
		} else {
			fmt.Fprint(w,
				`<HTML>
<HEAD>
<TITLE>portal</TITLE>
</HEAD>
<BODY>
<form action="login.go" method="post">
<table border="0">
<tr>
<td>用户名:</td> <td> <input type="text" name="UserName" /> </td>
</tr>
<tr>
<td>密码:</td> <td><input type="text" name="Password" /> </td>
</tr>
<tr>
<td></td>
<td><input type="submit" value="提交" /> </td>
<tr>
</form>
</BODY>
</HTML> `)
		}
		return
	}

	switch REDIRECT_MODE {
	default:
		// case 0: // 直接输出
		fmt.Fprint(w,
			`<HTML>
<HEAD>
<TITLE>portal</TITLE>
</HEAD>
<BODY>
<form action="login.go" method="post">
<table border="0">
<tr>
<td>用户名:</td> <td> <input type="text" name="UserName" /> </td>
</tr>
<tr>
<td>密码:</td> <td><input type="text" name="Password" /> </td>
</tr>
<tr>
<td></td>
<td><input type="submit" value="提交" /> </td>
<tr>
</form>
</BODY>
</HTML> `)

	case 1: // http/1.0 重定向
		// overwrite the client http version
		r.Proto = "HTTP/1.0"
		r.ProtoMajor = 1
		// r.ProtoMinor = 0

		if r.ProtoMinor == 0 {
			http.Redirect(w, r, "/login.go?mode=11", http.StatusSeeOther)
		} else {
			http.Redirect(w, r, "/login.go?mode=12", http.StatusSeeOther)
		}
	case 2: // http/1.1 重定向
		// overwrite the client http version
		r.Proto = "HTTP/1.1"
		r.ProtoMajor = 1
		r.ProtoMinor = 1

		http.Redirect(w, r, "/login.go?mode=2", http.StatusSeeOther)

	case 3: // javescript 跳转
		portalUrl := "http://" + FAKE_IP.To4().String() + "/login.go?mode=3"
		tmpl, err := template.New("foo").Parse(
			`<html>
<script language="javascript">
location.replace("{{.PortalUrl}}");
</script>
<body>
<p align="center">
<a href="{{.PortalUrl}}>">Welcome. If the browser cannot be redirected automatically, please click here, Thank you.</a>
</body>
</html>`)
		// type TemplateData struct {
		//   PortalUrl string
		// }
		if err != nil {
			fmt.Println("Template parse error. ", err)
			fmt.Fprint(w, "Template parse error.")
			return
		}
		err = tmpl.Execute(w, struct{ PortalUrl string }{PortalUrl: portalUrl})
		if err != nil {
			fmt.Println("Template execute error. ", err)
			fmt.Fprint(w, "Template execute error.")
		}

	case 4: // wispr
		// wispr specification 在网上已经找不到了。
		// 参考 https://github.com/wichert/wispr/blob/master/src/wispr/__init__.py 这里的代码大概猜测一下
		// https://docs.microsoft.com/en-us/windows-hardware/drivers/mobilebroadband/wispr-authentication
		wisprLoginUrl := "http://" + FAKE_IP.To4().String() + "/login.go"
		// 临时把method改一下，不然method == "GET" 的时候Redirect方法写一条链接进去网页内容里面去，我们不要写这个
		method := r.Method
		r.Method = "POST"
		http.Redirect(w, r, wisprLoginUrl, http.StatusFound)
		r.Method = method
		fmt.Fprint(w,
			`<HTML>
<!--
    <?xml version="1.0" encoding="UTF-8"?>
    <WISPAccessGatewayParam xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
			xsi:noNamespaceSchemaLocation="http://www.acmewisp.com/WISPAccessGatewayParam.xsd">
      <Redirect>
        <VersionHigh>1.0</VersionHigh>
        <VersionLow>0.0</VersionLow>
        <AccessProcedure>1.0</AccessProcedure>
        <AccessLocation>Hotel Guest Network</AccessLocation>
        <LocationName>Hotel Test</LocationName
        <LoginURL>`+wisprLoginUrl+`
        <MessageType>100</MessageType>
        <ResponseCode>0</ResponseCode>
      </Redirect>
    </WISPAccessGatewayParam>
-->
</HTML>`)

	case 5: // http head refresh 重定向   非标准，但大多浏览器都支持
		// req.Header.Add("If-None-Match", `W/"wyzzy"`)
		portalUrl := "http://" + FAKE_IP.To4().String() + "/login.go?mode=5"
		// w.Header().Set("Refresh", "0; url="+portalUrl)   // android  不支持这个，但支持下面的。IE 两个都支持
		fmt.Fprint(w, `<HTML><HEAD><TITLE>portal</TITLE><meta http-equiv="refresh" content="0; url=`+portalUrl+`"></HEAD></HTML>`)
	}
}

func PrintLocalIP() {
	ifaces, err := net.Interfaces()
	// handle err
	if err != nil {
		fmt.Println("获取不到本地ip地址\n")
		return
	}
	for _, i := range ifaces {
		addrs, err := i.Addrs()
		if err != nil {
			fmt.Println("获取不到本地ip地址\n")
			return
		}
		for _, address := range addrs {
			// check the address type and if it is not a loopback the display it
			if ipnet, ok := address.(*net.IPNet); ok && !ipnet.IP.IsLoopback() {
				if ipnet.IP.To4() != nil {
					fmt.Println(`可以通过这个地址访问 http:\\` + ipnet.IP.String())
				}
			}
		}
	}
}

//------------------------------------------------------------------------------

func main() {
	go func(msg string) {
		var h PortalServer
		// http.ListenAndServe("localhost:4000", &h)
		fmt.Println("\nhttp portal服务器已经启动")
		PrintLocalIP()
		http.ListenAndServe(":80", &h)
	}("http go coroutine")

	// ----------------------
	var servers []string
	args := flag.Args()

	if len(args) > 0 {
		servers = append(servers, args...)
	} else {
		servers = DEFAULT_SERVERS
	}

	LOG.Printf("Servers: %s", servers)

	proxyer := Proxy{servers}

	if err := dns.ListenAndServe("0.0.0.0:53", "udp", proxyer); err != nil {
		LOG.Fatal(err)
	}
}
```

这里http server和dns都写到一起了，这里只是让dns返回本地http的ip就可以，认证之前
所有的http都被域名劫持跳转到本地了。
测试android和iPhone都可以连wifi后自动正常跳出portal认证页面。





其他有用的测试手段
==================

1. 通过Xcode 或者iTools工具查看iPhone的syslog  （console log）
   测试之前，先安装iTools 4.0  （http://www.itools.cn/）
   然后iPhone 通过数据线和电脑连接，iTools里面选择 “实时日志”。
   然后再进行测试， 根据网上说法，登陆认证是CNA如果出错会有syslog记录。

2. 抓取iPhone的网络通讯包
  iOS Packet Tracing（利用usb链接iPhone和MAC电脑，然后使用rvictl在pc可以模拟出一个网络设备抓iPhone的网络包）
  https://developer.apple.com/library/content/qa/qa1176/_index.html#//apple_ref/doc/uid/DTS10001707-CH1-SECIOSPACKETTRACING

3. 上Apple develop 论坛（ Apple Developer Forums / Core OS / Networking）
   有个Apple的技术工程师叫做eskimo的好像对NEhotspothelper接口很了解，
   他留有邮箱可以向他咨询吧


参考文档：
------------------
Apple官方文档的“Hotspot Network Subsystem Programming Guide”里面的wifi认证状态机的说明
https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/Hotspot_Network_Subsystem_Guide/Contents/AuthStateMachine.html#//apple_ref/doc/uid/TP40016639-CH2-SW1

Apple的 Network Extenstion接口
https://developer.apple.com/reference/networkextension

https://forums.developer.apple.com/message/205571#205571
https://forums.developer.apple.com/message/223136#223136

