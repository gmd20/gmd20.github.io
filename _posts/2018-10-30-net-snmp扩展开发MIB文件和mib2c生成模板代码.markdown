# 首先要定义一个MIB，可以参考一下别的MIB，ASN.1 的定义
http://www.net-snmp.org/docs/mibs/


# 要把MIB文件加入到snmpd的配置目录里面去
http://www.net-snmp.org/tutorial/tutorial-5/toolkit/mib2c/index.html

# 查看系统snmp的MIB目录，把自己的XYZ-MIB.txt文件放到这个目录里面去
```text
[root@localhost snmp]# net-snmp-config --default-mibdirs
/root/.snmp/mibs:/usr/share/snmp/mibs
```

# 直接修改/etc/snmp/snmp.conf 或者个人目录的  ~/.snmp/snmp.conf 目录
mibfile FILE 或者 mibs MIBLIST 来执行要加载的MIB文件，参考man snmp.conf

# 也可以直接用MIBS变量指定MIB文件名

# 测试一下，自己定义的mib里面XYZTable是否能正常加载了
MIBS="+XYZ-MIB" snmptranslate -IR -Tp XYZTable

# mib2c 生成代码

```text
安装包： yum install net-snmp-perl
源代码： https://github.com/net-snmp/net-snmp/blob/master/local/mib2c

mib2c.array-user.conf
mib2c.create-dataset.conf
mib2c.int_watch.conf
mib2c.iterate.conf
mib2c.iterate_access.conf
mib2c.mfd.conf
mib2c.notify.conf
mib2c.old-api.conf
mib2c.scalar.conf
```

一般的table 使用 mib2c.mfd.conf这个就可以了, man mib2c 查看帮助
```text
MIBS="+XYZ-MIB" mib2c -c mib2c.mfd.conf XYZTable
MIBS="+XYZ-MIB" mib2c -c mib2c.mfd.conf -f <output_filename> XYZTable
```
/usr/bin/mib2c 是一个perl脚本，可以看一下

# 需要公司自定义 enterprise id 可以在这里申请
http://pen.iana.org/pen/PenApplication.page

# 修改snmpd.conf 加载 自定编译的模块
```text
dlmod test /usr/lib64/test.so
```

#  开启snmpagent的调试日志
```text
snmpd -Lf /tmp/snmpd.log  -D
```

# 测试扩展模块
```text
snmpwalk -v 1 localhost -c public system
snmpwalk -v 2c localhost -c public iftable
snmpwalk -v 2c 192.168.1.246  -c public  interfaces
MIBS="+XYZ-MIB" snmpwalk -v 2c localhost -c public enterprises.<申请到的或者测试用的id>

snmpget -v 2c 192.168.1.246 -c public .1.3.6.1.2.1.4.1.0
snmpgetnext -v 2c 192.168.1.246 -c public .1.3.6.1.2.1.4.33.0
snmpset -Os -OQ -Ou -c private -v 2c 192.168.1.246 enterprises.<申请到的或者测试用的id>.x.x.x i 0
···

# centos 7 上面netsnmp 5.7.2 的mib2c bug
```text

table类型的netsnmp_handler_registration_create，默认生成的 NETSNMP_NO_WRITE_SUPPORT宏的有问题，
HANDLER_CAN_RWRITE和HANDLER_CAN_RONLY可读写的判断刚好反过来了

        netsnmp_handler_registration_create("testTable", handler,
                                            testTable_oid,
                                            testTable_oid_size,
                                            HANDLER_CAN_BABY_STEP |
#if !(defined(NETSNMP_NO_WRITE_SUPPORT) || defined(NETSNMP_DISABLE_SET_SUPPORT))
                                            HANDLER_CAN_RWRITE
#elseHANDLER_CAN_RONLY
                                            HANDLER_CAN_RONLY
#endif                          /* NETSNMP_NO_WRITE_SUPPORT || NETSNMP_DISABLE_SET_SUPPORT */
···



