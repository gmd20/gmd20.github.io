



像iTools（http://www.itools.cn/） 这样的软件，可以使用usb数据线连接iPhone和PC之后，可以在PC上面通过他的“实时日志”功能，实时查看iPhone的syslog日志，可以看到系统或者应用的日志。
苹果的XCode开发平台是可以查看这个日志的。但这个iTools可以在windows平台查看比较方便一些，不是每个人都有苹果笔记本或者台式机来做开发啊。

iTools的原理是什么样的？ 在网上找了一下资料：

iPhone的这个system log日志，其实就是/var/log/syslog文件来，如果越狱之后很容易读取这个文件就可以了。
但没有越狱的机器需要使用iPhone内置的接口才行。

iPhone的PrivateFrameworks/MobileDevice.framework有个后台进程 “/usr/libexec/lockdownd” 可以提供
“com.apple.syslog_relay” 服务，通过iPhone的接口调用这个服务就可以了，其实任何lockdown的services.plist提供的服务都可以。


这有人提供一个OS X系统调用这个服务的例子
http://newosxbook.com/src.jl?tree=listings&file=jurpleConsole.c


这个MobileDevice的接口私有的，以前苹果都有开发windows 平台的 configure tool 可以用了读取console log数据。
不过好像现在不支持了？

不过网上有个开源项目了。说是跨平台Linux， Mac OS X 和 Windows都是可以。支持最新的iOS 10.
http://www.libimobiledevice.org/
https://github.com/libimobiledevice/libimobiledevice
这个库自带的读取iPhone的syslog的工具
https://github.com/libimobiledevice/libimobiledevice/blob/master/tools/idevicesyslog.c

有人编译好的windows平台的库：
https://github.com/Sn0wCooder/libimobiledevice-compiled-windows
https://github.com/rcmpayne/libimobiledevice-Compiled-Windows
后面这个说是支持ios 10的syslog， 参考
https://github.com/libimobiledevice/libimobiledevice/issues/325
https://github.com/libimobiledevice/libimobiledevice/issues/327

还有人说怎么在windows上面编译这个库：
http://docs.quamotion.mobi/en/latest/imobiledevice/compiling.html
windows的代码对应是下面的这些
https://github.com/libimobiledevice-win32/libplist
https://github.com/libimobiledevice-win32/libusbmuxd
https://github.com/libimobiledevice-win32/libimobiledevice
https://github.com/libimobiledevice-win32/ideviceinstaller


有了这个库，在windows平台调用就可以方便调用这个接口了，自己写个程序读取iPhone的日志
也是有可能的了。有时间再看看。其实要找这个东西是发现iTools的日志读取缓存太小了，
还不能直接保存到文件，日志比较多时都一下刷过去了，没能查看完整的日志。

这个libimobiledevice还是要依赖苹果的usb驱动才行的，所以使用时还是要预先安装苹果的iTunes，
它应该会把必须的驱动安装上。苹果现在应该不单独提供这些驱动的安装包了。
之前看iTools 安装完，它也是自动下载安装几个 “Apple mobile deive support”的包的，估计从
iTunes的里面剥离出来的吧。iTunes对应的iTunesMobileDevice.dll  文件说是在 C:\Program Files\Common Files\Apple\Mobile Device Support\iTunesMobileDevice.dll，64位系统在Program Files (x86)目录。还有CoreFoundation.dll等文件。
也可以直接根据头文件来调用里面的函数? (看网上说确实很多助手类软件都是使用iTunes的dll，然后用Loadlibrary动态加载方式来调用的。)

按照https://github.com/libimobiledevice/libusbmuxd 的说明，windows平台还是推荐使用iTunes的usbmuxd后台进程，iTunes应该搞定usb驱动和usb交换部分了。  这个libusbmuxd的只是通过socket接口和usbmuxd后台进程来进行交互而已。不过他们也有开发linux平台的https://github.com/libimobiledevice/usbmuxd
。windows平台还是用iTunes自带的那些比较好吧。iTunes应该自己在系统安装了驱动，然后会安装一个service “ Apple Mobile Device Service 
”， 估计就是这个usbmuxd的后台了。看libusbmuxd实现就是去连接本地的 127.0.0.1的27015这个端口，27015 应该就是usbmod的监听端口，苹果最新的iOS要求用TLS 安全连接去建立连接，通讯消息的格式好像就是plist封装，可以是二进制编码或者xml编码的吧。

刚刚试了在windows 10 + vc 2017编译libimobiledevice，很顺利， 不过要以来openssl 和 libiconv。有空试一下便宜出来的ideviceinfo 和idevicesyslog这两个命令看看怎么样


参考资料：
https://www.theiphonewiki.com/wiki/System_Log
https://www.theiphonewiki.com/wiki/MobileDevice_Library
https://stackoverflow.com/questions/7277804/ios-iphone-ipad-ipodtouch-view-real-time-console-log-terminal



