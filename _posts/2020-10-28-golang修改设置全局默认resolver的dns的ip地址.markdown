```golang
package main

import (
	"context"
	"fmt"
	"net"
)

func CloudflareDialer(ctx context.Context, network, address string) (net.Conn, error) {
	d := net.Dialer{}
	return d.DialContext(ctx, "udp", "1.1.1.1:53")
}

func main() {
	net.DefaultResolver.PreferGo = true
	net.DefaultResolver.Dial = CloudflareDialer
	fmt.Println(net.LookupHost("baidu.com"))
}

```
