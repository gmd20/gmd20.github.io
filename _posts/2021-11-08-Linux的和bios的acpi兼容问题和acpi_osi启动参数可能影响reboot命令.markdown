之前有遇到，bios里面“操作系统类型”设置会影响linux的reboot命令，不设置为“linux”类型，linux的reboot命令不起作用。

看网上很多帖子提到linux的重启问题和acpi_osi参数有关系，估计是bios的兼容问题吧。     

这篇官方解释比较详细：   
https://www.kernel.org/doc/html/latest/firmware-guide/acpi/osi.html   

完整的acpi参数参考， acpi_osi  acpi_os这两个比较相关吧，可以参考网上别人是怎么配置的。   
https://www.kernel.org/doc/html/v4.14/admin-guide/kernel-parameters.html#the-kernel-s-command-line-parameters


内核源码应该是这个    
https://github.com/torvalds/linux/blob/master/drivers/acpi/osi.c



