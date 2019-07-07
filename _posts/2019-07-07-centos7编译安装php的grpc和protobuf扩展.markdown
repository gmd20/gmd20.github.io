grpc php extension
==================

# 先按照官方文档编译grpc
```text
make install
```

# 编译grpc php extension
```text
yum -y install centos-release-scl.noarch
yum -y install rh-php72 rh-php72-php rh-php72-php-fpm
yum -y install rh-php72-php-devel

cd grpc/src/php/ext/grpc
/opt/rh/rh-php72/root/usr/bin/phpize
./configure --with-php-config=/opt/rh/rh-php72/root/usr/bin/php-config
make

[root@localhost grpc]# ldd .libs/grpc.so
	linux-vdso.so.1 =>  (0x00007fff097cd000)
	libgrpc.so.7 => /usr/local/lib/libgrpc.so.7 (0x00007f2c40f73000)
	libgpr.so.7 => /usr/local/lib/libgpr.so.7 (0x00007f2c40d66000)
	librt.so.1 => /lib64/librt.so.1 (0x00007f2c40b5e000)
	libdl.so.2 => /lib64/libdl.so.2 (0x00007f2c4095a000)
	libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f2c4073e000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f2c40371000)
	libssl.so.10 => /lib64/libssl.so.10 (0x00007f2c400ff000)
	libcrypto.so.10 => /lib64/libcrypto.so.10 (0x00007f2c3fc9d000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f2c3f99b000)
	libz.so.1 => /lib64/libz.so.1 (0x00007f2c3f785000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f2c41534000)
	libgssapi_krb5.so.2 => /lib64/libgssapi_krb5.so.2 (0x00007f2c3f538000)
	libkrb5.so.3 => /lib64/libkrb5.so.3 (0x00007f2c3f24f000)
	libcom_err.so.2 => /lib64/libcom_err.so.2 (0x00007f2c3f04b000)
	libk5crypto.so.3 => /lib64/libk5crypto.so.3 (0x00007f2c3ee18000)
	libkrb5support.so.0 => /lib64/libkrb5support.so.0 (0x00007f2c3ec08000)
	libkeyutils.so.1 => /lib64/libkeyutils.so.1 (0x00007f2c3ea04000)
	libresolv.so.2 => /lib64/libresolv.so.2 (0x00007f2c3e7eb000)
	libselinux.so.1 => /lib64/libselinux.so.1 (0x00007f2c3e5c4000)
	libpcre.so.1 => /lib64/libpcre.so.1 (0x00007f2c3e362000)
```


# 编译protobuf php extension
```text
protobuf-php-3.8.0.tar.gz
cd /root/protobuf-3.8.0/php/ext/google/protobuf
/opt/rh/rh-php72/root/bin/phpize
./configure --with-php-config=/opt/rh/rh-php72/root/usr/bin/php-config
make

[root@localhost protobuf]# ldd .libs/protobuf.so
	linux-vdso.so.1 =>  (0x00007fffe97d2000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f6a16da2000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f6a173d7000)
```



