
官方有一个修改注册表的库。   
https://pkg.go.dev/golang.org/x/sys/windows/registry

修改windows系统代理，要修改注册表的这两个键
CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\ProxyEnable  
CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\ProxyServer  
 
参考
https://blog.csdn.net/Jack_he_li/article/details/111467607

