像ping -I 命令一样指定 ip 或者网卡名字，golang里面指定源ip是比较方便的，但指定源接口作为出口就不太方便了。
需要 为Dialer的Control里面对原始fd进行原始的syscall系统调用。看了源码研究了好一会才好到换个方法

```go
        s.tcpDialer = &net.Dialer{
                LocalAddr: tcpSource,
                Timeout:   30 * time.Second,
                KeepAlive: 30 * time.Second,
        }

        if bindToDevice {
                ctrlFn := func(network, address string, c syscall.RawConn) error {
                        c.Control(func(fd uintptr) {
                                err := syscall.BindToDevice(int(fd), ifname)
                                if err != nil {
                                        fmt.Printf("bind to device %s err: %s\n", uc.Source, err.Error())
                                }
                        })
                        return nil
                }
                s.tcpDialer.Control = ctrlFn
        }

```
