这个好像只能代理http，不能https，估计socks5那种才能处理https ？
设置dns服务器地址的只有在linux下面才会用golang 的resolver才会有效果，windows下面不使用这个设置。
```golang
package main

import (
	"context"
	"flag"
	"log"
	"net"
	"net/http"
	"net/http/httputil"
	"time"
)

func main() {
	var dnsServer string
	var localIp string

	flag.StringVar(&dnsServer, "dns_server", "", "dns server to use(1.1.1.1:53)")
	flag.StringVar(&localIp, "local_ip", "", "local ip address for http request")
	flag.Parse()

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

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		log.Print("11", r.URL)

		proxy.ServeHTTP(w, r)
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}

```
