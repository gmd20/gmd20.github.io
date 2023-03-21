windows 默认好不支持 http basicauth，只支持https，好像需要改一下注册表才行。
但如果不用密码的话，直接“映射网络驱动器”是可以的。

```go
package main

import (
	"flag"
	"net/http"

	"golang.org/x/net/webdav"
)

func main() {
	flagRootDir := flag.String("dir", "/", "webdav root dir")
	flagHttpAddr := flag.String("address", ":8888", "listen address")
	flagHttpsMode := flag.Bool("https-mode", false, "use https mode")
	flagCertFile := flag.String("https-cert-file", "cert.pem", "https cert file")
	flagKeyFile := flag.String("https-key-file", "key.pem", "https key file")
	flagUserName := flag.String("username", "", "username")
	flagPassword := flag.String("password", "", "password")
	flagReadonly := flag.Bool("read-only", false, "read only")

	flag.Parse()
	fs := &webdav.Handler{
		FileSystem: webdav.Dir(*flagRootDir),
		LockSystem: webdav.NewMemLS(),
	}
	http.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
		if *flagUserName != "" && *flagPassword != "" {
			username, password, ok := req.BasicAuth()
			if !ok {
				w.Header().Set("WWW-Authenticate", `Basic realm="Restricted"`)
				w.WriteHeader(http.StatusUnauthorized)
				return
			}
			if username != *flagUserName || password != *flagPassword {
				http.Error(w, "WebDAV: need authorized!", http.StatusUnauthorized)
				return
			}
		}
		if *flagReadonly {
			switch req.Method {
			case "PUT", "DELETE", "PROPPATCH", "MKCOL", "COPY", "MOVE":
				http.Error(w, "WebDAV: Read Only!!!", http.StatusForbidden)
				return
			}
		}
		fs.ServeHTTP(w, req)
	})
	if *flagHttpsMode {
		http.ListenAndServeTLS(*flagHttpAddr, *flagCertFile, *flagKeyFile, nil)
	} else {
		http.ListenAndServe(*flagHttpAddr, nil)
	}
}
```
