发现有的系统cat /proc/cpuinfo里面的“cpu MHz” 比实际的低，原来除了睿频的，cpu还支持低功耗模式，也会降频的，
可以看 /sys/devices/system/cpu/cpu0/cpufreq/ 这里面的输出，好像intel的有的cpu不支持睿频但可以支持几个步进的频率这种。

```text
/sys/module/ipses/parameters # cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_
min_freq 
1332800
/sys/module/ipses/parameters # cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_
max_freq 
1999200
/sys/module/ipses/parameters # cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_
cur_freq 
1999688

/sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
```

centos 8 上面有一个 cpupower命令 可以用来设置修改主频
```text
NAME
       cpupower-frequency-set - A small tool which allows to modify cpufreq settings.

SYNTAX
       cpupower [ -c cpu ] frequency-set [options]

DESCRIPTION
       cpupower  frequency-set allows you to modify cpufreq settings without having to type e.g.
       "/sys/devices/system/cpu/cpu0/cpufreq/scaling_set_speed" all the time.
       cpupower [ -c cpu ] frequency-set [options]

```

这个cpufreq好像是还有几个默认模式的governor， 到底是最求性能还是省电模式。

```text
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors 
performance powersave

/ # cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
performance

/ # cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
powersave

echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
设置了性能模式后，马上就可以看到CPU当前的工作主频升上去了。不过好像这个好像和cpu有关，有的在powersave模式，cpu 20% 了好像主频也不会自动升上去。
#  cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
2799829
```

内核相关驱动在 kernel/drivers/cpufreq 这个目录下吧


下面这个文章提到intel默认应该是powersave模式，但一般都会自动提高主频的。
https://wiki.archlinux.org/index.php/CPU_frequency_scaling
```text
Depending on the scaling driver, one of these governors will be loaded by default:

ondemand for AMD and older Intel CPU.
powersave for Intel CPUs using the intel_pstate driver (Sandy Bridge and newer).
Note: The intel_pstate driver supports only the performance and powersave governors, but they both provide dynamic scaling. The performance governor should give better power saving functionality than the old ondemand governor.
```

