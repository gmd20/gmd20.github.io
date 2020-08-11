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

```

centos 8 上面有一个 cpupower 可以用来设置修改主频
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
/ # cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
performance
/ # cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
powersave

echo powersave > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```
内核相关驱动在 kernel/drivers/cpufreq 这个目录下吧
