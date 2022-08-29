
官方有一个修改注册表的库。   
https://pkg.go.dev/golang.org/x/sys/windows/registry

修改windows系统代理，要修改注册表的这两个键
HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\ProxyEnable   整型类型
HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\ProxyServer   字符串类型
 
参考
https://blog.csdn.net/Jack_he_li/article/details/111467607

