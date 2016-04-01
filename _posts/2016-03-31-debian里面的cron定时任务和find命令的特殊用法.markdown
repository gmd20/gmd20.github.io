cron 定时任务
============
https://debian-handbook.info/browse/zh-CN/stable/sect.task-scheduling-cron-atd.html

设置任务
--------

```bash
export EDITOR=vim
crontab -u username -e
```
```text
# 创建每隔一分钟运行的任务
*/1 * * * * date >> /opt/test/log/cron.log

# 创建每天凌晨4点1分运行的任务
1 4 * * * date >> /opt/test/log/cron.log
```

find命令的特殊用法
==================
```text
查找/opt/temp 目录下的 名字扩展名为gz 修改时间在1分钟之前的文件 把找到的第1个匹配文件名作为 输入参数，调用/opt/collector.sh 来处理。  -quit表示找到第一个文件就退出。如果不指定-quit 则是对每个文件都进行处理.  -execdir和 -quit是 and 关系，也就是是前面execdir命令返回成功了才会执行 quit，参考 man find里面 对expression的解释。
find /opt/temp -name '*.gz' -mmin +1  -execdir /opt/collector.sh {} \; -quit

重复执行两次，这样也能达到只处理两个文件的目的，当然前提是第一次处理的时候就把文件改名了，不然第一个find还是返回一个一样的结果吧
find /opt/temp  \( -name '*.gz' -mmin +1  -execdir /opt/collector.sh {} \; \) ,  \( -name '*.gz' -mmin +1  -quit \) && find /opt/temp  \( -name '*.gz' -mmin +1  -execdir /opt/collector.sh {} \; \) ,  \( -name '*.gz' -mmin +1  -quit \) 

删除所有最后修改时间为一分钟之前的gz文件
find /opt/temp -name '*.gz' -mmin +1  -delete 

删除所有最后修改时间一天之前的gz文件
find /opt/temp -name '*.gz' -mtime +1  -delete 

显示所有最后修改时间在1分钟之前的目录
find /opt/temp -type d  -mmin +1

删除所有的空目录和文件
find /opt/temp  -type d -empty -delete
```


