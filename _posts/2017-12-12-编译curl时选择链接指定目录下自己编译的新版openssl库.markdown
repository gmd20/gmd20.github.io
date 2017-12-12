自己从源码编译一个新的openssl
=============================
下载源码下来解压到 /home/ming/openssl-1.0.2m

```text
./config -h
/usr/bin/perl ./Configure  -h

./config shared --prefix=/usr/mylibs --openssldir=/usr/mylibs/ssl
make clean
make
make install

make install 之后它会把头文件和动态库静态库都复制到/usr/mylibs目录下，
系统旧版的0.98版本的openssl和头文件那些都在系统目录。怕系统其他依赖，
不想破坏系统文件
```




编译curl，链接新的openssl
=========================
这一个很麻烦，curl的congfure脚本老是使用系统旧版的openssl，

```text
export PKG_CONFIG_PATH=/usr/mylibs/lib/pkgconfig
pkg-config --cflags --libs libssl

./configure --prefix=/usr/mylibs --with-ssl=/home/ming/openssl-1.0.2m \
  --disable-ldap --disable-ldaps --disable-rtsp --disable-proxy --disable-dict \
  --enable-tftp --disable-pop3 --disable-imap --disable-smb --disable-smtp \
  --disable-gopher --disable-manual --disable-ipv6 --disable-verbose \
  --disable-sspi --disable-crypto-auth --disable-ntlm-wb \
LDFLAGS="-Wl,-rpath-link=/usr/mylibs/lib -Wl,--verbose"

用--with-ssl=/usr/mylibs来配置是不行，可能由于系统已经存在0.98e的openssl，老是
认出旧版本的openssl，连头文件包含目录都没有配置，配置了PKG_CONFIG_PATH也不行。
使用--with-ssl=/home/ming/openssl-1.0.2m 的话，可以看到Makefile里面是把
--with-ssl=/home/ming/openssl-1.0.2m做为openssl的目录了，头文件正确了，
但LDFLAGS = -L/home/ming/openssl-1.0.2m/lib的链接库还是不对的，需要源码目录
/home/ming/curl-7.57.0/lib/Makefile 里面的LDFLAGS改为 LDFLAGS = -L/usr/mylibs/lib
然后再看他ld搜索的路径优先使用 -L/usr/mylibs/lib 目录下面的 libssl.so 了。


因为自己编译的openssl（libssl.so）安装在/usr/mylibs/lib 目录下面，不用-rpath/-rpath-link指定
的话，ld还是优先会使用系统自带的旧版/usr/lib/libssl.so。
--rpath-link来指定具体要链接的动态库的目录，但看上去不起作用，这个只有在编译exe的
的时候才起作用？编译so不起作用？还是ld版本太旧的原因？
尝试LD_PRELOAD， LD_LIBRARY_PATH  和 /etc/ld.so.conf 这些也是没有什么用的。

不过好在-L指定的目录顺序是排在/lib /usr/lib系统目录的前面，这样才链接到了正确版本的libssl.so
通过这个 -Wl,--verbose 参数才看出问题了。
```

编译php使用新的curl和openssl
===========================
依赖openssl和curl

```text
/home/ming/php-5.6.32

./configure --prefix=/opt --without-mysql --without-sqlite3 --without-pdo-sqlite \
--without-iconv --disable-xmlreader --disable-xmlwriter --with-pcre-regex \
--enable-sockets --without-pear --disable-debug --disable-fileinfo \
--with-openssl=/usr/mylibs  --with-curl=/usr/mylibs \
LDFLAGS="-Wl,-rpath-link=/usr/mylibs/lib -Wl,--verbose"

检查Makefile里面的CFLAGS没有包含-I/usr/kerberos/include 和
LDFLAGS不包含-L/usr/kerberos/lib 旧版的openssl库
rpath看起来也是正常的 -Wl,-rpath,/usr/mylibs/lib


编译链接正常
[root@ming php-5.6.32]# ldd sapi/cli/php
	linux-gate.so.1 =>  (0x00c12000)
	libcrypt.so.1 => /lib/libcrypt.so.1 (0x00556000)
	libresolv.so.2 => /lib/libresolv.so.2 (0x0058f000)
	libcrypto.so.1.0.0 => /usr/mylibs/lib/libcrypto.so.1.0.0 (0x0014b000)
	libm.so.6 => /lib/libm.so.6 (0x002e9000)
	libdl.so.2 => /lib/libdl.so.2 (0x00110000)
	libnsl.so.1 => /lib/libnsl.so.1 (0x00361000)
	libxml2.so.2 => /usr/lib/libxml2.so.2 (0x0092c000)
	libz.so.1 => /usr/lib/libz.so.1 (0x00114000)
	libcurl.so.4 => /usr/mylibs/lib/libcurl.so.4 (0x00ac8000)
	libssl.so.1.0.0 => /usr/mylibs/lib/libssl.so.1.0.0 (0x00eaa000)
	librt.so.1 => /lib/librt.so.1 (0x00356000)
	libc.so.6 => /lib/libc.so.6 (0x00378000)
	/lib/ld-linux.so.2 (0x0012f000)
	libpthread.so.0 => /lib/libpthread.so.0 (0x00310000)

[root@ming php-5.6.32]# ldd sapi/cgi/php-cgi
	linux-gate.so.1 =>  (0x00318000)
	libcrypt.so.1 => /lib/libcrypt.so.1 (0x00556000)
	libresolv.so.2 => /lib/libresolv.so.2 (0x0058f000)
	libcrypto.so.1.0.0 => /usr/mylibs/lib/libcrypto.so.1.0.0 (0x0014b000)
	libm.so.6 => /lib/libm.so.6 (0x002e9000)
	libdl.so.2 => /lib/libdl.so.2 (0x00110000)
	libnsl.so.1 => /lib/libnsl.so.1 (0x00361000)
	libxml2.so.2 => /usr/lib/libxml2.so.2 (0x0092c000)
	libz.so.1 => /usr/lib/libz.so.1 (0x00114000)
	libcurl.so.4 => /usr/mylibs/lib/libcurl.so.4 (0x00319000)
	libssl.so.1.0.0 => /usr/mylibs/lib/libssl.so.1.0.0 (0x0067d000)
	librt.so.1 => /lib/librt.so.1 (0x00378000)
	libc.so.6 => /lib/libc.so.6 (0x00381000)
	/lib/ld-linux.so.2 (0x0012f000)
	libpthread.so.0 => /lib/libpthread.so.0 (0x004c4000)

```


总结：系统存在多版本时的链接问题
========================
如果系统在两个地方有两个不同版本的库，要选择链接使用某一个时，还是用 ld连接器的  -L参数来制定库的目录。   
最好还是直接修改Makefile。 其他configure脚本 pkg-config 环境变量等不是都不可靠，还是直接修改Makefile   
才行。  
当然能保证兼容性问题，之安装一个版本，尽量用系统提供的安装包和头文件那是最好的了。

