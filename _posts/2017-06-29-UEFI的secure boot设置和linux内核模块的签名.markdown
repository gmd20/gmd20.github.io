26.8. SIGNING KERNEL MODULES FOR SECURE BOOT
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/System_Administrators_Guide/sect-signing-kernel-modules-for-secure-boot.html

Loading signed kernel modules
https://lwn.net/Articles/470906/

Managing EFI Boot Loaders for Linux: Dealing with Secure Boot
http://www.rodsbooks.com/efi-bootloaders/secureboot.html

Kernel module signing facility
https://www.kernel.org/doc/html/v4.10/admin-guide/module-signing.html

SecureBoot
https://wiki.ubuntu.com/SecurityTeam/SecureBoot

Ubuntu-kernel/Documentation/module-signing.txt
https://github.com/Canonical-kernel/Ubuntu-kernel/blob/master/Documentation/module-signing.txt

"Secure Boot" 是UEFI标准里面用来防病毒的吧，用来阻止恶意引导程序的加载。 UEFI里面预先保存有受信任
的公钥，比如OEM厂家的或者微软的公钥。有的好像个人也可以自己配置一些公钥（Machine Owner Key (MOK)）进去。
如果这个选项被设置了，像grub这些引导程序的efi镜像就必须经过 公钥对应的私钥的签名才能加载了。
所以有的时候安装系统时，想要加载没签名果的grub或者grub模块，比较禁用bios里面的这个“secure boot”了。

Linux内核模块签名机制“Kernel module signing”，也会在加载模块检查 *.ko 文件里面的签名，这个签名一般
保存在elf文件格式的一个section里面吧。  自己编译内核的时候，linux的编译脚本会自己生成一个公钥私钥对，
用来给编译的内核模块签名。make install的时候，应该就用私钥给模块做签名了。这些都是自动，如果内核和
模块都是自己编译的，也不太需要关心这个问题。 甚至可以自己修改内核代码把这个签名校验机制给删掉都可以的，
好像以前工作的一个公司的就是这么干的，甚至把内核的模块版本和内核版本匹配的检查都去掉了。好像也有几个相关的内核配置选项。

但ubuntu 和redhat这些发行版，看他们的文档，如果他们发现uefi里面的secure boot是打开的，运行时候linux
内核里面会获取的到UEFI里面的system_keyring，那内核也就启动签名校验，用这uefi传过来的公钥校验模块的签名。
这样有一个什么问题呢，就是第三方驱动的内核模块，一般不是ubuntu这些编译的，是后来才下载编译的，编译的时候
不可能拿到ubuntu官方的私钥来做签名。这些不签名的驱动，在 “secure boot” 模式下面就会被内核拒接加载了。
比如自己编译的Intel wireless adapter driver，一些nVidia的显卡驱动等等。 要用这种第三方驱动只能把bios里面
的secure boot给关了，这样内核也不检查模块的签名了。要么就要自己给自己编译的模块签名，还要把自己使用的
私钥对应的公钥加到 UEFI的数据库里面去。看前面的问题，有可能一个bios可以自己配置或者有MOK相关的工具可以
修改UEFI的公钥数据库。


