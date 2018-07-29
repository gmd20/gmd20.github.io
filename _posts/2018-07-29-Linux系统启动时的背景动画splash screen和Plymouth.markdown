启动时的静态的背景图是grub里面提供的功能，设置一个图片就可以了，自己搜索 grub splash screen可以搜索到很多相关的配置。
参考 http://www.tuxfixer.com/set-grub2-custom-splash-screen-on-rhel-7-centos-7-uefi-and-legacy-bios-iso-image/




像centos和ubuntu等动画式启动进度条的，应该都是通过PLYMOUTH 这个来软件来实现的。这个软件允许配置之制作动画式的主题背景。可以查看哥哥发行版的文档，
这个应该利用了linux的framebuffer显卡驱动来实现的，有个后台进程在不停的刷显示缓存。



https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/desktop_migration_and_administration_guide/plymouth
https://wiki.ubuntu.com/Plymouth
