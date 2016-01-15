看到有文章说postgresql的最大响应延时时间，cfq的最差。好像一般来说ssd磁盘也推荐使用费cfq的io调度器。

http://blog.pgaddict.com/posts/postgresql-io-schedulers-cfq-noop-deadline
http://blog.pgaddict.com/posts/postgresql-with-different-io-schedulers

Linux I/O tuning: CFQ vs. deadline
http://www.postgresql.org/message-id/flat/4B6FDD26.9010106@2ndquadrant.com#4B6FDD26.9010106@2ndquadrant.com

What is the recommended I/O scheduler for a database workload in Red Hat Enterprise Linux?
https://access.redhat.com/solutions/54164

PostgreSQL pain points
https://lwn.net/Articles/591723/

在自己的机器上面也测试一下看看：
测试环境：
vmware esxi
debian7 32bit
ext4
普通的机械硬盘
iostat -x 4 显示
         w\s   ：100左右，
         r\s   ：几乎都是0
         %util ： 80% 左右
         %iowait ： 4% ~ 6%
对同一个表insert /update /delete    测试时运行insert的tps为600， update和 delete应该也差不多


在这个压力下分别运行 cfq ，deadline， noop的io scheduler1个小时
```
#!/bin/bash

date +%s
date
echo deadline > /sys/block/sda/queue/scheduler
cat  /sys/block/sda/queue/scheduler
echo "---------------------------------"

sleep 3600

date +%s
date
echo cfq > /sys/block/sda/queue/scheduler
cat  /sys/block/sda/queue/scheduler
echo "---------------------------------"

sleep 3600

date +%s
date
echo noop > /sys/block/sda/queue/scheduler
cat  /sys/block/sda/queue/scheduler
echo "---------------------------------"

sleep 3600

echo "done"

```


测试结果发现确实使用cfg的时候耗时超长的insert出现的机会更大一些。 但不知道这个测试时间1个小时足够说明问题没有。


