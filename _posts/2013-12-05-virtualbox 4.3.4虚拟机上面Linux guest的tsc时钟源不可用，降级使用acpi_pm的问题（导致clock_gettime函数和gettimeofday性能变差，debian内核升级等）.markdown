```text
测试发现，程序的时间获取函数clock_gettime 导致内核路径的acpi_pm_read 函数代价很大。
---------------------------------------------
perf top -e cpu-cycles  -G -c 1 -p 4120


-  38.17%  [kernel]                   [k] acpi_pm_read                                                                                         
   - acpi_pm_read                                                                                                                              
      - 57.13% 0xb76e94                                                                                                                        
           31.44% fd_cnx_s_sendv                                                                                                               
      - 36.44% clock_gettime                                                                                                                   
           52.90% fd_fifo_post_internal                                                                                                        
           18.61% mq_pop                                                                                                                       
           13.97% process_thr                                                                                                                  
           4.91% fd_psm_next_timeout                                                                                                           
           3.32% fd_p_sr_store                                                                                                                 
           3.20% fd_cnx_s_sendv                                                                                                                
           3.08% sr_expiry_th                                                                                                                  
        0.88% pthread_cond_signal@@GLIBC_2.3.2                                                                                                 
        0.82% __memcpy_ssse3                                                                                                                   
-  12.10%  [kernel]                   [k] arch_local_irq_enable                                                                                
   - arch_local_irq_enable                                                                                                                     
        35.05% 0xb76e94                                                                                                                        
        16.79% clock_gettime                                                                                                                   
        11.46% pthread_cond_signal@@GLIBC_2.3.2                                                                                                
        5.16% __strlen_sse2_bsf 
        4.76% __memcpy_ssse3                                                                                                                   
        2.51% __strcmp_ssse3                                                                                                                   
        2.02% process_cin_item                                                                                                                 
        1.35% _int_malloc                                                                                                                      
        1.29% vfprintf                                                                                                                         
        1.16% free                                                                                                                             
        0.94% __memset_sse2                                                                                                                    
        0.65% __memcmp_ssse3                                                                                                                   
        0.59% _int_free                                                                                                                        
-  11.99%  [kernel]                   [k] arch_local_irq_restore                                                                               
   - arch_local_irq_restore                                                                                                                    
      + 46.99% 0xb76e94                                                                                                                        
        45.65% pthread_cond_signal@@GLIBC_2.3.2                                                                                                
        0.82% clock_gettime                                                                                                                    
        0.73% __strlen_sse2_bsf                                                                                                                
        0.67% __memcpy_ssse3                                                                                                                   
-   6.50%  [e1000]                    [k] e1000_irq_enable                                                                                     
   - e1000_irq_enable                                                                                                                          
        70.74% 0xb76e94                                                                                                                        
        3.79% __memcpy_ssse3                                                                                                                   
        2.90% pthread_cond_signal@@GLIBC_2.3.2                                                                                                 
        2.25% clock_gettime                                                                                                                    
        2.16% __strlen_sse2_bsf                                                                                                                
        1.39% process_cin_item                                                                                                                 
        1.33% __strcmp_ssse3                                                                                                                   
        0.99% _int_malloc                                                                                                                      
        0.90% free                                                                                                                             
        0.63% __memcmp_ssse3                                                                                                                   
        0.58% vfprintf                                                                                                                         
        0.54% _int_free                                                                                                                        
-   6.31%  [e1000]                    [k] e1000_xmit_frame   




root@debian02:/# perf top -e task-clock  -G -c 10 -p 4010

Events: 4M task-clock, Thread: freeDiameterd(4010)
+  43.67%  [kernel]                   [k] acpi_pm_read
+  11.87%  [e1000]                    [k] e1000_irq_enable
+  10.67%  [e1000]                    [k] e1000_xmit_frame
+   9.58%  [kernel]                   [k] arch_local_irq_restore 
+   6.86%  libc-2.13.so               [.] 0x705a8
+   3.52%  [e1000]                    [k] e1000_alloc_rx_buffers
+   3.38%  [kernel]                   [k] arch_local_irq_enable
+   0.51%  libcee_origin.so           [.] PCU_GET_CIN
+   0.33%  libc-2.13.so               [.] free
+   0.33%  libc-2.13.so               [.] malloc
+   0.23%  libcee_origin.so           [.] CIN_CHAR_APPEND
+   0.21%  libstdc++.so.6.0.17        [.] std::string::compare(std::string const&) const
+   0.20%  libcee_origin.so           [.] CIN_LONG_APPEND
+   0.18%  [kernel]                   [k] arch_local_irq_enable

-------------------------------------------
看上去是因为程序里面很多地方调用 clock_gettime函数 ，然后clock_gettime 又调用了 acpi_pm_read  导致的。
可以看到acpi_pm_read函数的消耗非常的大。这一方面是 clock_gettime确实比较多。但想这种时间函数，和gettimeofday
相关以前看过资料介绍说是有vdiso优化，应该很快的啊。
这个 acpi_pm_read 应该对应的是ACPI_PM时钟源，性能要比TSC的差的，不知道这个virtualbox虚拟机上面为什么默认使用的acpi_pm而不是tsc的clock来源。

这里提一下，应用程序做超时判断，应该使用 clock_gettime 函数的CLOCK_MONOTONIC参数来而不是CLOCK_REALTIME参数，以获取单调增长的时间作为程序的参数判断才行。
避免时间回绕导致clock时间变小，导致的超时判断不正确的问题。



http://man7.org/linux/man-pages/man2/clock_gettime.2.html
网上有人分析clock_gettime的实现  http://www.bluezd.info/archives/clock_gettime


source/kernel/posix-timers.c
source/kernel/time/timekeeping.c

273 /*
274  * Initialize everything, well, just everything in Posix clocks/timers ;)
275  */
276 static __init int init_posix_timers(void)
277 {
278         struct k_clock clock_realtime = {
279                 .clock_getres   = hrtimer_get_res,
280                 .clock_get      = posix_clock_realtime_get,
281                 .clock_set      = posix_clock_realtime_set,
282                 .clock_adj      = posix_clock_realtime_adj,

 posix_clock_realtime_get -》ktime_get_real_ts-》
getnstimeofday -》__getnstimeofday-> timekeeping_get_ns 



166 static inline s64 timekeeping_get_ns(struct timekeeper *tk)
167 {
168         cycle_t cycle_now, cycle_delta;
169         struct clocksource *clock;
170         s64 nsec;
171 
172         /* read clocksource: */
173         clock = tk->clock;
174         cycle_now = clock->read(clock);     ///调用时钟源的对应函数


acpi_pm是的源码是这么注册时钟源的
source/drivers/clocksource/acpi_pm.c
clocksource_register_hz(&clocksource_acpi_pm,MTMR_TICKS_PER_SEC)

 66 static struct clocksource clocksource_acpi_pm = {
 67         .name           = "acpi_pm",
 68         .rating         = 200,
 69         .read           = acpi_pm_read,                /////////////这个就是我们最前面看到perf里面的函数
 70         .mask           = (cycle_t)ACPI_PM_MASK,
 71         .flags          = CLOCK_SOURCE_IS_CONTINUOUS,
 72 };



source/kernel/time/clocksource.c
这个有个全局的clocksource_list链表，所有的注册其实就是加到这个链表里面来而已。

下面的/sys文件系统查看时钟源的功能是这两个函数导出的
sysfs_show_current_clocksources
sysfs_show_available_clocksources

-----------------------
source/arch/x86/vdso/vclock_gettime.c
         __vdso_clock_gettime
         __vdso_gettimeofday

如果clock_gettime 使用 CLOCK_REALTIME参数，那么跟gettimeofday函数一样底层都是调用的
do_realtime 这个函数来。


http://stackoverflow.com/questions/14314535/where-do-the-stack-vdso-and-vsyscall-mmaps-come-from
这里有一些介绍
They are created automatically by the kernel when is loads a file into memory to run it.

The precise control flows for [vdso] and [vsyscall] are hard to follow because there is all sorts of defining and redefining of function names as macros depending on whether the kernel is 32 or 64 bit but some relevant routines include:

load_elf_binary in fs/binfmt_elf.c which calls arch_setup_additional_pages
arch_setup_additional_pages in arch/x86/vdso/vma.c
arch_setup_additional_pages in arch/x86/vdso/vdso32-setup.c
The [stack] mapping is not ELF specific and is created by __bprm_mm_init in fs/exec.c which is called by the execve code before it invokes the format specific loader.



arch/x86/vdso/vma.c

arch/x86/vdso/vdso32-setup.c

arch/x86/vdso/vclock_gettime.c

arch/x86/kernel/vsyscall_64.c

kernel/time/timekeeping.c

相关的源码上面这几个比较重要吧，但还有很多链接脚本和汇编之类的文件。

时钟源更新时间的时候，应该是通过这个 timekeeping_update -》 update_vsyscall，来更新vclock_gettime.c里面访问的vsyscall_gtod_data这个变量。然后vclock_gettime.c里面就可以直接使用这个缓存最新的时间了，只在有必要的时候才会访问硬件获取最新时间。


这个大体流程大概了解一些了，但要弄清楚，还要看timekeeping 模块的，还有内核初始化时候的time_init  tsc_init等函数。这个vdsio里面好像是没有acpi_pm的timer的读取的。
估计acpi_pm还是走的普通的syscall接口了。

--------------------------




Chapter 15. Timestamping  15.1. Hardware clocks
https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_MRG/2/html/Realtime_Reference_Guide/chap-Realtime_Reference_Guide-Timestamping.html
根据上面这篇文章
可以知道，内核在启动的时候，会从众多时钟源里面选择一个，首选Time Stamp Counter (TSC),其次是High Precision Event Timer (HPET)，
当前面都不可用的才会去用另外的 ACPI Power Management Timer (ACPI_PM) ， Programmable Interval Timer (PIT) ， Real Time Clock (RTC)
。但PIT和 RTC说是精度不够或者代价太大的吧。

根据前面那篇文章，  TSC的性能大概比HPET好20倍，TSC性能比ACPI_PM要好40倍。 但很遗憾virtuabox里面的debian guest竟然使用的是最差的ACPI_PM。


Linux内核的clocksource可以通过接口，看到当前使用的 是哪一种和可以用的都有哪一些的。

--------
检查一下Virualbox的虚拟机的时钟源
root@debian01:/# cat /sys/devices/system/clocksource/clocksource0/current_clocksource
acpi_pm
root@debian01:/# 
root@debian01:/# cat /sys/devices/system/clocksource/clocksource0/available_clocksource
acpi_pm 

---------
另外一个是vmware的
root@TEST-LINUX-01:~# cat /sys/devices/system/clocksource/clocksource0/current_clocksource
tsc
root@TEST-LINUX-01:~# cat /sys/devices/system/clocksource/clocksource0/available_clocksource
tsc hpet acpi_pm 
--------------

vmware的3种都可以使用，但virualbox只有acpi_pm 可以使用。

系统使用的时钟源应该是可以修改的，像下面这样。
echo hpet > /sys/devices/system/clocksource/clocksource0/current_clocksource

------------------------------
http://www.virtualbox.org/manual/ch09.html#changetscmode
virtualbox的设置，只看到一个可以调整TSC的选项，但没有其他相关的了。

自己测试一下。
看到两个修改可以影响到默认时间源的选择的：
1.
virtulbox 的系统设置里面把 主板芯片从PIIX3改为ICH9，那么hpet acpi_pm 两个都可以使用，然后默认会使用hpet

2. 
把guest的cpu个数配置为1，那么tsc hpet acpi_pm 3种都可以使用。默认会选tsc。
这个应该是一个cpu之后，Linux内核里面检查tsc时钟源是否可用的代码可以通过了。后面会说。不需要多个cpu之间的同步检测了。
但cpu数量少了，很影响guest的性能，完全不能接受的修改吧。



---------------------
对比virtualbox和vmware启动的时候TSC时钟源的检测。

root@debian01:/# dmesg |grep TSC
[    0.000000] Fast TSC calibration using PIT
[    0.144008] TSC synchronization [CPU#0 -> CPU#1]:
[    0.144008] Measured 66930 cycles TSC warp between CPUs, turning off TSC clock.
[    0.144008] Marking TSC unstable due to check_tsc_sync_source failed

root@EST-LINUX-01:~# dmesg |grep TSC
[    0.000000] TSC freq read from hypervisor : 2394.000 MHz
[    0.222151] Skipped synchronization checks as TSC is reliable.

可以看到一个检测的时候，cpu之间同步检测 check_tsc_sync_source失败了， 另外一个成功认出vmware hypervisor ，跳过了检查。


root@debian01:/# cat /proc/cpuinfo | grep tsc
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3


可以看到cpu其实都是支持tsc的，而且constant_tsc这个应该是Intel为了在多cpu之间的TSC不准确问题而做的改进特性？
但虚拟机经过了这么虚拟机一层，可能同一个cpu会被多个guest分享等原因。这个TSC时钟源的检查在Linux内核里面检查受影响通不过。
很奇怪vmware为啥可以使用TSC呢，在网上搜索到这个
https://lkml.org/lkml/2008/10/20/469

看文章的解释，大概是专门为vmware的hypervior加一个补丁，
在
arch/x86/kernel/tsc.c
文件的
get_hypervisor_tsc_freq 
函数。
专门在这里绕过正常Linux TSC时钟源的检查了，应该在虚拟机上面这个是很容易出问题的。
这就是为什么vmware内拿到tsc时钟源的原因。
virtualbox应该也要自己加上类似这个功能才行，不然还是没法使用TSC时钟源的。


最新的内核3.12的代码应该是source/arch/x86/kernel/cpu/vmware.c 这个里面的
vmware_platform_setup  ->vmware_get_tsc_khz 函数


但还是没看到virtualbox的专门的代码来做这个事情的。估计还是没法使用了。



抱着最后的希望找找看有没有内核参数相关的，找到这个。

https://www.kernel.org/doc/Documentation/kernel-parameters.txt


	clock=		[BUGS=X86-32, HW] gettimeofday clocksource override.
			[Deprecated]
			Forces specified clocksource (if available) to be used
			when calculating gettimeofday(). If specified
			clocksource is not available, it defaults to PIT.
			Format: { pit | tsc | cyclone | pmtmr }

	clocksource=	Override the default clocksource
			Format: <string>
			Override the default clocksource and use the clocksource
			with the name specified.
			Some clocksource names to choose from, depending on
			the platform:
			[all] jiffies (this is the base, fallback clocksource)
			[ACPI] acpi_pm
			[ARM] imx_timer1,OSTS,netx_timer,mpu_timer2,
				pxa_timer,timer3,32k_counter,timer0_1
			[AVR32] avr32
			[X86-32] pit,hpet,tsc;
				scx200_hrt on Geode; cyclone on IBM x440
			[MIPS] MIPS
			[PARISC] cr16
			[S390] tod
			[SH] SuperH
			[SPARC64] tick
			[X86-64] hpet,tsc


	notsc		[BUGS=X86-32] Disable Time Stamp Counter

	tsc=		Disable clocksource stability checks for TSC.
			Format: <string>
			[x86] reliable: mark tsc clocksource as reliable, this
			disables clocksource verification at runtime, as well
			as the stability checks done at bootup.	Used to enable
			high-resolution timer mode on older hardware, and in
			virtualized environment.
			[x86] noirqtime: Do not use TSC to do irq accounting.
			Used to run time disable IRQ_TIME_ACCOUNTING on any
			platforms where RDTSC is slow and this accounting
			can add overhead.

通过设置 tsc=reliable 这个参数用来禁用内核的时钟源稳定检测，跳过启动时候的 TSC稳定检查。 看样子就是为了专门在这个虚拟机里面强制使用TSC时钟源的。


==========================
马上来验证一下；

修改
/etc/default/grub 
给
GRUB_CMDLINE_LINUX_DEFAULT
加上  tsc=reliable

update-grub2 
更新配置后，检查
/boot/grub/grug.cfg
里面，参数已经加上去了


-----------
在ubuntu server 12.04  3.8内核 可以看到tsc检查被跳过了，
dmesg里面有这个输出
Skipped synchronization checks as TSC is reliable

检查系统时钟源已经使用了TSC的了。

但在debian 3.2内核里面设置这个参数还是不起作用。






tsc_clocksource_reliable  这个整型变量对应上面那个内核参数，在tsc_setup函数里面设置的。

source/arch/x86/kernel/tsc_sync.c
----------------------3.2的内核的代码-----
101 /*
102  * Source CPU calls into this - it waits for the freshly booted
103  * target CPU to arrive and then starts the measurement:
104  */
105 void __cpuinit check_tsc_sync_source(int cpu)
106 {
107         int cpus = 2;
108 
109         /*
110          * No need to check if we already know that the TSC is not
111          * synchronized:
112          */
113         if (unsynchronized_tsc())
114                 return;
115 
116         if (boot_cpu_has(X86_FEATURE_TSC_RELIABLE)) {
117                 if (cpu == (nr_cpu_ids-1) || system_state != SYSTEM_BOOTING)
118                         pr_info(
119                         "Skipped synchronization checks as TSC is reliable.\n");
120                 return;
121         }


---------------------3.8内核的代码-------------


119 
120 /*
121  * Source CPU calls into this - it waits for the freshly booted
122  * target CPU to arrive and then starts the measurement:
123  */
124 void __cpuinit check_tsc_sync_source(int cpu)
125 {
126         int cpus = 2;
127 
128         /*
129          * No need to check if we already know that the TSC is not
130          * synchronized:
131          */
132         if (unsynchronized_tsc())
133                 return;
134 
135         if (tsc_clocksource_reliable) {
136                 if (cpu == (nr_cpu_ids-1) || system_state != SYSTEM_BOOTING)
137                         pr_info(
138                         "Skipped synchronization checks as TSC is reliable.\n");
139                 return;
140         }


-------------------------------



source/arch/x86/kernel/cpu/vmware.c 
vmware里面会自己设置这个cpu的特性的。
110 /*
111  * VMware hypervisor takes care of exporting a reliable TSC to the guest.
112  * Still, due to timing difference when running on virtual cpus, the TSC can
113  * be marked as unstable in some cases. For example, the TSC sync check at
114  * bootup can fail due to a marginal offset between vcpus' TSCs (though the
115  * TSCs do not drift from each other).  Also, the ACPI PM timer clocksource
116  * is not suitable as a watchdog when running on a hypervisor because the
117  * kernel may miss a wrap of the counter if the vcpu is descheduled for a
118  * long time. To skip these checks at runtime we set these capability bits,
119  * so that the kernel could just trust the hypervisor with providing a
120  * reliable virtual TSC that is suitable for timekeeping.
121  */
122 static void __cpuinit vmware_set_cpu_features(struct cpuinfo_x86 *c)
123 {
124         set_cpu_cap(c, X86_FEATURE_CONSTANT_TSC);
125         set_cpu_cap(c, X86_FEATURE_TSC_RELIABLE);
126 }
127 
128 const __refconst struct hypervisor_x86 x86_hyper_vmware = {
129         .name                   = "VMware",
130         .detect                 = vmware_platform,
131         .set_cpu_features       = vmware_set_cpu_features,
132         .init_platform          = vmware_platform_setup,
133 };
134 EXPORT_SYMBOL(x86_hyper_vmware);


3.2和3.8的check_tsc_sync_source函数不太一样，如果使用tsc_clocksource_reliable这个变量那么就可以支持tsc=reliable参数了。
这个在3.3版本就开始支持了。可怜debian是3.2版本，还是没能把这个tsc的时钟源用起来。

d


尝试升级一下debian的内核
======================



参考backport的说明
http://backports.debian.org/Instructions/

修改 /etc/apt/sources.list文件，给我debian7.0 的发行版 wheezy 加上面类似这么一句
deb http://YOURMIRROR.debian.org/debian wheezy-backports main

我用的是
deb http://mirrors.ustc.edu.cn/debian/ wheezy-backports  main

然后更新数据库
apt-get update

查看一下backport的kernel
apt-cache search linux-image


决定使用最新的3.11的
linux-image-3.11-0.bpo.2-686-pae
linux-headers-3.11-0.bpo.2-686-pae
linux-image-3.11-0.bpo.2-686-pae-dbg


apt-get -t wheezy-backports install linux-image-3.11-0.bpo.2-686-pae
apt-get -t wheezy-backports install linux-headers-3.11-0.bpo.2-686-pae         //virualbox-guest会有编译问题 
apt-get -t wheezy-backports install linux-image-3.11-0.bpo.2-686-pae-dbg
apt-get -t wheezy-backports install linux-tools-3.11         /// perf工具在这个包里面

root@debian01:/home/bright# update-grub2 
Generating grub.cfg ...
Found background image: /usr/share/images/desktop-base/desktop-grub.png
Found linux image: /boot/vmlinuz-3.11-0.bpo.2-686-pae
Found initrd image: /boot/initrd.img-3.11-0.bpo.2-686-pae
Found linux image: /boot/vmlinuz-3.2.0-4-686-pae
Found initrd image: /boot/initrd.img-3.2.0-4-686-pae
done


----------------------------
bright@debian01:~$ cat /proc/cmdline 
BOOT_IMAGE=/boot/vmlinuz-3.11-0.bpo.2-686-pae root=UUID=ff84db67-7d11-4732-ba0c-8681536dedd4 ro tsc=reliable quiet
bright@debian01:~$ cat /sys/devices/system/clock
clockevents/ clocksource/ 
bright@debian01:~$ cat /sys/devices/system/clocksource/clocksource0/available_clocksource 
tsc acpi_pm 
bright@debian01:~$ cat /sys/devices/system/clocksource/clocksource0/current_clocksource 
tsc
bright@debian01:~$ uname -a
Linux debian01 3.11-0.bpo.2-686-pae #1 SMP Debian 3.11.8-1~bpo70+1 (2013-11-21) i686 GNU/Linux

终于使用了tsc 时钟了。 至于有没有什么副作用没有就不清楚了。





重新运行文章开始的freeDiameter的perf性能测试
===========================================
apt-get -t wheezy-backports install linux-tools-3.11   //perf工具在这个包里面
perf top -e cpu-cycles  -G -c 1 -p 3512


-  44.81%  [kernel]                   [k] read_tsc                                                                                             
   - read_tsc                                                                                                                                  
      - 53.32% 0xb7762424                                                                                                                      
           13.89% fd_cnx_s_sendv                                                                                                               
           1.29% 0x100100c0                                                                                                                    
           0.68% 0xa6b00040                                                                                                                    
           0.60% 0xb5a00040                                                                                                                    
         + 0.60% 0xa6900040                                                                                                                    
      - 44.15% clock_gettime                                                                                                                   
         + 56.67% fd_fifo_post_internal                                                                                                        
         + 15.80% process_thr                                                                                                                  
         + 11.36% mq_pop                                                                                                                       
           7.56% fd_cnx_s_sendv                                                                                                                
         + 5.28% fd_sess_new                                                                                                                   
         + 2.07% fd_psm_next_timeout                                                                                                           
           1.27% exp_fct                                                                                                                       
              __clone                                                                                                                          
+  13.35%  [e1000]                    [k] e1000_xmit_frame                                                                                     
+   8.84%  [e1000]                    [k] e1000_clean                                                                                          
+   8.40%  [kernel]                   [k] _raw_spin_unlock_irqrestore                                                                          
+   3.14%  [kernel]                   [k] __do_softirq                                                                                         
+   2.72%  [e1000]                    [k] e1000_alloc_rx_buffers                                                                               
+   1.90%  libc-2.13.so               [.] __memcpy_ssse3                                                                                       
+   1.44%  libc-2.13.so               [.] __strcmp_ssse3                                                                                       
+   1.11%  libcee_origin.so           [.] PCU_GET_CIN                                                                                          
+   1.11%  libc-2.13.so               [.] __strlen_sse2_bsf                                                                                    
+   0.90%  libc-2.13.so               [.] _int_malloc                                                                                          
+   0.90%  [kernel]                   [k] hrtimer_peek_ahead_timers                                                                            
+   0.51%  libcee_origin.so           [.] CIN_CHAR_APPEND   



可以看到这个时间函数还在不过变成了  read_tsc  函数。
还是占用非常的高啊。 看来这个freeDiameter要改进一下时间函数的使用才行，后面要给官方反馈一下这个问题。
不过系统的整体性能应该提升了，外面测试request响应时间缩短很多。 top -H 观察到的cpu也下降了很多。





2013-12-13进一步补充

根据测试virtualbox 设置cpu个数是1的时候，非常快。 多个cpu 的时候就非常慢

有人已经注意到这个问题了
https://www.virtualbox.org/ticket/11892
https://www.virtualbox.org/changeset/45436/vbox
https://www.virtualbox.org/browser/vbox/trunk/src/VBox/VMM/VMMR3/TM.cpp

多cpu 的tsc不同步的bug
https://www.virtualbox.org/ticket/7915

http://en.wikipedia.org/wiki/Time_Stamp_Counter

vmware关于虚拟tsc等时钟源的解释。
Timekeeping in VMware Virtual Machines 
http://www.vmware.com/files/pdf/Timekeeping-In-VirtualMachines.pdf

KVM的相关的文档
http://docs.fedoraproject.org/en-US/Fedora/13/html/Virtualization_Guide/chap-Virtualization-KVM_guest_timing_management.html



导致这个问题的代码的解释
346	346	        else 
347	347	            pVM->tm.s.fMaybeUseOffsettedHostTSC = true; 
 	348	        /** @todo needs a better fix, for now disable offsetted mode for VMs 
 	349	         * with more than one VCPU. With the current TSC handling (frequent 
 	350	         * switching between offsetted mode and taking VM exits, on all VCPUs 
 	351	         * without any kind of coordination) it will lead to inconsistent TSC 
 	352	         * behavior with guest SMP, including TSC going backwards. */ 
 	353	        if (pVM->cCpus != 1) 
 	354	            pVM->tm.s.fMaybeUseOffsettedHostTSC = false; 
348	355	    } 



We're aware that this change has consequences. However it was the only realistic short term fix for seriously buggy TSC behavior in the SMP case, because without this change TSC could go backwards 

temporarily, violating the most important specification how TSC is supposed to operate: as a monotonically increasing counter. This "going backwards" misbehavior confuses some software extremely, which is 

why backing out this change is simply not an option. Correctness has priority over performance in this case, however we're planning to develop a better fix.







新的i7 cpu是支持constant_tsc特性的，多cpu的tsc的计数保证单调递增，这样就不会有所谓的smp 环境的tsc回绕问题了吧。
bright@debian01:~$ cat /proc/cpuinfo |grep constant_tsc
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp constant_tsc pni ssse3
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht nx rdtscp 
constant_tsc pni ssse3




网上有人提到的intel的手册的介绍
X86_FEATURE_CONSTANT_TSC + X86_FEATURE_NONSTOP_TSC bits in cpuid (edx=x80000007, bit #8; check unsynchronized_tsc function of linux kernel for more checks)

Intel's Designer's vol3b, section 16.11.1 Invariant TSC it says the following

"16.11.1 Invariant TSC

The time stamp counter in newer processors may support an enhancement, referred to as invariant TSC. Processor's support for invariant TSC is indicated by CPUID.80000007H:EDX[8].

The invariant TSC will run at a constant rate in all ACPI P-, C-. and T-states. This is the architectural behavior moving forward. On processors with invariant TSC support, the OS may use the TSC for wall 

clock timer services (instead of ACPI or HPET timers). TSC reads are much more efficient and do not incur the overhead associated with a ring transition or access to a platform resource."

So, if TSC can be used for wallclock, they are guaranteed to be in sync.





virtualbox的几个虚拟tsc相关的几个设置项
   /*
 * Determine the TSC configuration and frequency.
 */
/* mode */
/** @cfgm{/TM/TSCVirtualized,bool,true}
 * Use a virtualize TSC, i.e. trap all TSC access. */
rc = CFGMR3QueryBool(pCfgHandle, "TSCVirtualized", &pVM->tm.s.fTSCVirtualized);
if (rc == VERR_CFGM_VALUE_NOT_FOUND)
	pVM->tm.s.fTSCVirtualized = true; /* trap rdtsc */
else if (RT_FAILURE(rc))
	return VMSetError(pVM, rc, RT_SRC_POS,
					  N_("Configuration error: Failed to querying bool value \"UseRealTSC\""));

/* source */
/** @cfgm{/TM/UseRealTSC,bool,false}
 * Use the real TSC as time source for the TSC instead of the synchronous
 * virtual clock (false, default). */
rc = CFGMR3QueryBool(pCfgHandle, "UseRealTSC", &pVM->tm.s.fTSCUseRealTSC);
if (rc == VERR_CFGM_VALUE_NOT_FOUND)
	pVM->tm.s.fTSCUseRealTSC = false; /* use virtual time */
else if (RT_FAILURE(rc))
	return VMSetError(pVM, rc, RT_SRC_POS,
					  N_("Configuration error: Failed to querying bool value \"UseRealTSC\""));
if (!pVM->tm.s.fTSCUseRealTSC)
	pVM->tm.s.fTSCVirtualized = true;

/* TSC reliability */
/** @cfgm{/TM/MaybeUseOffsettedHostTSC,bool,detect}
 * Whether the CPU has a fixed TSC rate and may be used in offsetted mode with
 * VT-x/AMD-V execution.  This is autodetected in a very restrictive way by
 * default. */
rc = CFGMR3QueryBool(pCfgHandle, "MaybeUseOffsettedHostTSC", &pVM->tm.s.fMaybeUseOffsettedHostTSC);
if (rc == VERR_CFGM_VALUE_NOT_FOUND)
{
	if (!pVM->tm.s.fTSCUseRealTSC)
		pVM->tm.s.fMaybeUseOffsettedHostTSC = tmR3HasFixedTSC(pVM);
	else
		pVM->tm.s.fMaybeUseOffsettedHostTSC = true;
	/** @todo needs a better fix, for now disable offsetted mode for VMs
	 * with more than one VCPU. With the current TSC handling (frequent
	 * switching between offsetted mode and taking VM exits, on all VCPUs
	 * without any kind of coordination) it will lead to inconsistent TSC
	 * behavior with guest SMP, including TSC going backwards. */
	if (pVM->cCpus != 1)                        //////////////////多个cpu时禁用快速tsc方式
		pVM->tm.s.fMaybeUseOffsettedHostTSC = false;
}

/** @cfgm{TM/TSCTicksPerSecond, uint32_t, Current TSC frequency from GIP}
 * The number of TSC ticks per second (i.e. the TSC frequency). This will
 * override TSCUseRealTSC, TSCVirtualized and MaybeUseOffsettedHostTSC.
 */
rc = CFGMR3QueryU64(pCfgHandle, "TSCTicksPerSecond", &pVM->tm.s.cTSCTicksPerSecond);
if (rc == VERR_CFGM_VALUE_NOT_FOUND)
{
	pVM->tm.s.cTSCTicksPerSecond = tmR3CalibrateTSC(pVM);
	if (    !pVM->tm.s.fTSCUseRealTSC
		&&  pVM->tm.s.cTSCTicksPerSecond >= _4G)
	{
		pVM->tm.s.cTSCTicksPerSecond = _4G - 1; /* (A limitation of our math code) */
		pVM->tm.s.fMaybeUseOffsettedHostTSC = false;
	}
}
else if (RT_FAILURE(rc))
	return VMSetError(pVM, rc, RT_SRC_POS,
					  N_("Configuration error: Failed to querying uint64_t value \"TSCTicksPerSecond\""));
else if (   pVM->tm.s.cTSCTicksPerSecond < _1M
		 || pVM->tm.s.cTSCTicksPerSecond >= _4G)
	return VMSetError(pVM, VERR_INVALID_PARAMETER, RT_SRC_POS,
					  N_("Configuration error: \"TSCTicksPerSecond\" = %RI64 is not in the range 1MHz..4GHz-1"),
					  pVM->tm.s.cTSCTicksPerSecond);
else
{
	pVM->tm.s.fTSCUseRealTSC = pVM->tm.s.fMaybeUseOffsettedHostTSC = false;
	pVM->tm.s.fTSCVirtualized = true;
}

/** @cfgm{TM/TSCTiedToExecution, bool, false}
 * Whether the TSC should be tied to execution. This will exclude most of the
 * virtualization overhead, but will by default include the time spent in the
 * halt state (see TM/TSCNotTiedToHalt). This setting will override all other
 * TSC settings except for TSCTicksPerSecond and TSCNotTiedToHalt, which should
 * be used avoided or used with great care. Note that this will only work right
 * together with VT-x or AMD-V, and with a single virtual CPU. */
rc = CFGMR3QueryBoolDef(pCfgHandle, "TSCTiedToExecution", &pVM->tm.s.fTSCTiedToExecution, false);
if (RT_FAILURE(rc))
	return VMSetError(pVM, rc, RT_SRC_POS,
					  N_("Configuration error: Failed to querying bool value \"TSCTiedToExecution\""));
if (pVM->tm.s.fTSCTiedToExecution)
{
	/* tied to execution, override all other settings. */
	pVM->tm.s.fTSCVirtualized = true;
	pVM->tm.s.fTSCUseRealTSC = true;
	pVM->tm.s.fMaybeUseOffsettedHostTSC = false;
}






virtualbox的文档没看到对其他几个参数的介绍。但前面的vmware的文档解释的比较清楚。商业的就是比这种开源的在这里啊。

按照我的理解，因为以前硬件多个cpu的时候，tsc时钟在不同的cpu上面不是很稳定，在不同的cpu上面不能保证tick数是单调递增的。
所以导致软件出问题，很多软件要想方设法避开这个问题。这里的virtubox多cpu里面的禁止MaybeUseOffsettedHostTSC就是这样，导致多cpu
上面的virtual tsc性能很差。 （根据测试打开慢10倍左右，clock_gettime从200微秒到2000微秒的差距）。这样对那些虚拟机运行的频繁获取时间的
程序有比较大的影响。但现在的cpu，主流的方向都是保证tsc计数在不同的cpu上面也都是单调递增的了，这样就没有这个问题了。
我的这个3代i7就有 constant_tsc  标志了。这样使用用MaybeUseOffsettedHostTSC 应该是足够安全的吧，可以获取得到更好的性能。


参考 http://www.virtualbox.org/manual/ch09.html#warpguest的设置
强制启用MaybeUseOffsettedHostTSC，应该是vcpu统计时，把vcpu切换出去的时间计数排除在外吧。这样rdtsc返回的时间可能就显得小一些。
VBoxManage setextradata "VM name" "VBoxInternal/TM/MaybeUseOffsettedHostTSC" 1
取消设置
VBoxManage setextradata "VM name" "VBoxInternal/TM/MaybeUseOffsettedHostTSC"

 使用原始tsc作为时钟源来计算吧，不是使用它自己虚拟tsc的维护的不同cpu同步时间
VBoxManage setextradata "VM name" "VBoxInternal/TM/UseRealTSC" 1
取消设置
VBoxManage setextradata "VM name" "VBoxInternal/TM/UseRealTSC" 

禁用虚拟TSC，不知道是不是类似的vmware文档介绍的禁止对rdtsc做虚拟化？
C:\Program Files\Oracle\VirtualBox>VBoxManage setextradata "debian_01" "VBoxInte
rnal/TM/TSCVirtualized" 0

实际操作测试一下

C:\Program Files\Oracle\VirtualBox>VBoxManage.exe --help

C:\Program Files\Oracle\VirtualBox>VBoxManage.exe setextradata "debian_01" "VBoxnternal/TM/MaybeUseOffsettedHostTSC" 1
C:\Program Files\Oracle\VirtualBox>VBoxManage setextradata "debian_01" "VBoxInternal/TM/UseRealTSC" 1

C:\Program Files\Oracle\VirtualBox>VBoxManage setextradata "debian_01" "VBoxInte
rnal/TM/TSCVirtualized" 0

检查 D:\VirtualBox VMs\debian_01\debian_01.vbox 这个xml可以看到下面这个，说明已经设置成功了。
<ExtraDataItem name="VBoxInternal/TM/MaybeUseOffsettedHostTSC" value="1"/>
<ExtraDataItem name="VBoxInternal/TM/UseRealTSC" value="1"/>
启动后D:\VirtualBox VMs\debian_01\logs\VBox.log里面也有这几个值的记录。


启动虚拟机，重新执行clock_gettime函数的测试，

bright@debian01:~/test$ ./clock_gettime_test 
CLOCK_REALTIME:
	 run 20000000 times,  total 4467601141 nanoseconds, avarage 223 nanoseconds

CLOCK_REALTIME_COARSE:
	 run 20000000 times,  total 4027102630 nanoseconds, avarage 201 nanoseconds

CLOCK_MONOTONIC:
	 run 20000000 times,  total 4432737422 nanoseconds, avarage 221 nanoseconds

CLOCK_MONOTONIC_COARSE:
	 run 20000000 times,  total 4097287108 nanoseconds, avarage 204 nanoseconds

total = 1790461225


结果看起来非常爽啊， 赶紧给所有的虚拟机都设置一下。有没有什么副作用还要后面观察一下。
但不确定这3个设置对性能有没有影响。因为这个clock_gettime 返回的是rdtsc的时间，如果使用MaybeUseOffsettedHostTSC返回的偏移值就会显的小一些。
不过按照vmware的文档，如果不使用virtual tsc，不对rdtsc做虚拟化，而是使用原始tsc，那么计数会准确一些，比如用来做性能测试的时间衡量基准，以前是推荐频繁调用这个时间获取函数的程序使用原始rdtsc的， 性能会更好一些。但可能导致vcpu测量时间不是很准确?有的时候可能会出问题
```
