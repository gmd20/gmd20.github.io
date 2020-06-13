https比http的特殊，用的connect tunnel的技术。
设置dns服务器地址的只有在linux下面才会用golang 的resolver才会有效果，windows下面不使用这个设置。

```golang
package main

import (
	"context"
	"flag"
	"io"
	"log"
	"net"
	"net/http"
	"net/http/httputil"
	"time"
)

var dnsServer string
var localIp string

type ReverseProxy struct{}

func (h *ReverseProxy) ProxyHTTPS(rw http.ResponseWriter, req *http.Request) {
	hij, ok := rw.(http.Hijacker)
	if !ok {
		log.Printf("http server does not support hijacker")
		return
	}

	clientConn, _, err := hij.Hijack()
	if err != nil {
		log.Printf("http: proxy error: %v", err)
		return
	}

	d := net.Dialer{}
	if len(localIp) > 0 {
		d.LocalAddr = &net.TCPAddr{
			IP:   net.ParseIP(localIp),
			Port: 0,
		}
	}

	if len(dnsServer) > 0 {
		r := &net.Resolver{
			PreferGo: true,
			Dial: func(ctx context.Context, network, address string) (net.Conn, error) {
				d := net.Dialer{
					Timeout: time.Millisecond * time.Duration(10000),
				}
				return d.DialContext(ctx, "udp", dnsServer)
			},
		}
		d.Resolver = r
	}
	proxyConn, err := d.Dial("tcp", req.URL.Host)
	if err != nil {
		log.Printf("http: proxy error: %v", err)
		return
	}

	deadline := time.Now()
	deadline = deadline.Add(time.Second * 30)

	err = clientConn.SetDeadline(deadline)
	if err != nil {
		log.Printf("http: proxy error: %v", err)
		return
	}
	err = proxyConn.SetDeadline(deadline)
	if err != nil {
		log.Printf("http: proxy error: %v", err)
		return
	}

	_, err = clientConn.Write([]byte("HTTP/1.0 200 OK\r\n\r\n"))
	if err != nil {
		log.Printf("http: proxy error: %v", err)
		return
	}

	go func() {
		io.Copy(clientConn, proxyConn)
		clientConn.Close()
		proxyConn.Close()
	}()

	io.Copy(proxyConn, clientConn)
	proxyConn.Close()
	clientConn.Close()
}

func (h *ReverseProxy) ProxyHTTP(w http.ResponseWriter, r *http.Request) {
	director := func(req *http.Request) {
		log.Print("222", req.URL)
	}
	proxy := &httputil.ReverseProxy{Director: director}

	if len(dnsServer) > 0 || len(localIp) > 0 {
		var r *net.Resolver
		var addr *net.TCPAddr

		if len(dnsServer) > 0 {
			r = &net.Resolver{
				PreferGo: true,
				Dial: func(ctx context.Context, network, address string) (net.Conn, error) {
					d := net.Dialer{
						Timeout: time.Millisecond * time.Duration(10000),
					}
					return d.DialContext(ctx, "udp", dnsServer)
				},
			}
		}

		if len(localIp) > 0 {
			addr = &net.TCPAddr{
				IP:   net.ParseIP(localIp),
				Port: 0,
			}
		}

		t := &http.Transport{
			Proxy: http.ProxyFromEnvironment,
			DialContext: (&net.Dialer{
				Timeout:   30 * time.Second,
				KeepAlive: 30 * time.Second,
				DualStack: true,
				Resolver:  r,
				LocalAddr: addr,
			}).DialContext,
			ForceAttemptHTTP2:     true,
			MaxIdleConns:          100,
			IdleConnTimeout:       90 * time.Second,
			TLSHandshakeTimeout:   10 * time.Second,
			ExpectContinueTimeout: 1 * time.Second,
		}
		proxy.Transport = t
	}

	proxy.ServeHTTP(w, r)
}

func (h *ReverseProxy) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	log.Println("", r.URL)

	if r.Method == "CONNECT" {
		h.ProxyHTTPS(w, r)
	} else {
		h.ProxyHTTP(w, r)
	}
}

func main() {
	var bindAddr string
	flag.StringVar(&bindAddr, "bind", "127.0.0.1:1080", "bind address")
	flag.StringVar(&dnsServer, "dns_server", "", "dns server to use(1.1.1.1:53)")
	flag.StringVar(&localIp, "local_ip", "", "local ip address for http request")
	flag.Parse()

	var h ReverseProxy
	log.Fatal(http.ListenAndServe(bindAddr, &h))
}
```


windows 10的网络设置里面代理好像是不支持socks5代理的，只支持http代理。
需要用开源的privoxy来把http代理转成socks5代理吧, 比如下面这个配置privoxy监听本地的127.0.0.1:1080端口，然后把流量转到
本地的127.0.0.1:8080端口。   

cat  privoxy_config.txt   
```text
listen-address 127.0.0.1:1080
toggle 0
logfile privoxy.log
show-on-task-bar 0
activity-animation 0
forward-socks5 / 127.0.0.1:8080 .
max-client-connections 2048
hide-console
```
启动
```text
./privoxy.exe  privoxy_config.txt
```


常用的socks5代理有 ssh -D  还有  https://github.com/shadowsocks/go-shadowsocks2  这种加密socks5通道等
```text
ssh -D 8080  -p 22 root@<ip-address>
```
