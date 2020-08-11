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
