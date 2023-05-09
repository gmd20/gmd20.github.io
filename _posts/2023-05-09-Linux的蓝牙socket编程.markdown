# 关键字  
bluez libbluetooth AF_BLUETOOTH  RFCOMM   SDP

#  参考资料

1. “An Introduction to Bluetooth Programming”  Albert Huang   
https://people.csail.mit.edu/albert/bluez-intro/   
讲了基本概念和c语言例子代码都有  


2. Bluez的文档   
https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc/mgmt-api.txt   
linux基本都要安装bluez这个包了，基本的蓝牙协议栈框架了

3. Android的蓝牙传输框架，包装好的接口使用应该是比较方便了   
https://developer.android.com/guide/topics/connectivity/bluetooth/connect-bluetooth-devices   


4. golang的RFCOMM接口   
Bluetooth Serial Port with Go and BlueZ   
https://www.christopherng.org/posts/bluetooth-serial-port-with-go-and-bluez/   

5. golang语言这里也有一个简单的rfcomm socket接口的例子   
https://pkg.go.dev/golang.org/x/sys/unix#SockaddrRFCOMM   

6. rfcomm命令可以把蓝牙通道映射为一个字符设备供应用程序使用
https://manpages.debian.org/bullseye/bluez/rfcomm.1.en.html

# 总结
蓝牙的设备的初始化和UUID服务初始化要bluez，RFCOMM 是蓝牙协议里面一个子模块，是模拟串口传输serial port 功能的，使用这个可以在设别直接创建socket接口双发收发3数据。
