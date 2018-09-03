# 1. 反编译apk文件为资源文件和smali汇编

java -jar apktool_2.3.3.jar  d -f test.apk -o test-apktool  有的应用用最新apktool_2.3.3版本的解压不了
java -jar apktool_2.2.4.jar  d -f test.apk -o test-apktool  换成apktool_2.2.4老版本可以反汇编成功。

java -jar apktool_2.3.3.jar  d -f app-release-unsigned.apk -o test_apk 自己构建apk的用apktool_2.3.3也可以解压

网上有人说有的开发者针对apktool的漏洞，估计构建特殊的apk让apktool反汇编不了，
如果遇到错误可以试一下不同版本
https://bitbucket.org/iBotPeaches/apktool/downloads/

# 2. apk 改名为zip，然后解压出classes.dex 源码文件

classes.dex是android的java虚拟机的指令吧，apktool好像也是使用的dex-tools来反编译的。
dex-tools 可以直接把dex文件转成java的class文件，然后用java decompiler就可以反编译得到java源代码。
cd dex-tools-2.1-SNAPSHOT
dex-tools-2.1-SNAPSHOT>d2j-dex2jar.bat classes.dex

 
# 3. jd-gui 打开jar查看源码

java -jar jd-gui-1.4.0.jar

jd-gui-windows-1.4.0/jd-gui.exe

但好像jar在 java 1.8的运行不起来。

# 4.  jar解压后，里面的class文件就是java的字节码了，

把class文件拖到android studio
的intellij的IDE里面，可以直接反汇编得到java代码，不过有时会报告错误。
可以查看smali汇编文件里面的源文件名字。
https://source.android.com/devices/tech/dalvik/dalvik-bytecode

发现混淆过的文件smali里面还是有以前的源代码文件名字的，可以用grep简单的找到对应的类名。
grep source -r ./smali |grep java  > ../smali_classes.txt
grep source -r ./smali_classes2 |grep java  > ../smali_classes2.txt


# 5. 修改代码

结合class的源码，可以手工修改smali汇编代码，smali代码看起来还挺简单的，可以看一下网上别人的介绍和android的文档。


# 6、重新打包apk

先删除原始apk目录META-INF签名文件夹的指纹文件 MANIFEST.MF *.SF *.RSA
java -jar apktool_2.2.4.jar b test-apktool  test-new.apk  会在 test-apktool/dist目录里面看到生成的apk。不过重新打包的比以前大很多，很是奇怪。


# 7. 签名

在java的bin目录加入path变量里面， 比如我用的java_cmd.bat 终端
```text
@echo off

set PATH=%PATH%;%cd%;C:\Program Files\Android\Android Studio\jre\bin

cmd.exe

```
然后在java终端执行下面两个命令生成key和签名就行了。
keytool -genkey -alias abc.keystore -keyalg RSA -validity 20000 -keystore abc.keystore
jarsigner -verbose -keystore abc.keystore -signedjar test_signed.apk dingding_new.apk abc.keystore


看网上的介绍，Android 的签名分为v1和v2两种签名，也可以同时使用两种签名的， android 7之后必须使用v2。
v1的签名应该是jar的签名机制，就是上面命令了。 v2的好像说是要使用Android sdk的 apksigner 命令。
不过看android studio 好像是使用这个 
C:\Program Files\Android\Android Studio\gradle\m2repository\com\android\tools\build\apksig\3.1.4\apksig-3.1.4.jar


试了v1的签名在我自己手机的android 5上面还是能用的。 aptool会把META-INF目录全部删掉了，但我看原始的apk除了MANIFEST.MF *.SF *.RSA这三个签名文件，还有其他文件的
。我是等apktool构建出来apk了，再手工改成zip文件，然后把原始META-INF目录的几个文件又加回去。
一方折腾之后，重新打包的apk能够安装成功，运行也可以。不过这个应用加了防破解机制，好像运行起来会检查签名，检测是被别人修改过的版本了，不能工作了。
修改的smali的地方还是没改对。











