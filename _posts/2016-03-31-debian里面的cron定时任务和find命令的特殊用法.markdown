cron 定时任务
============
https://debian-handbook.info/browse/zh-CN/stable/sect.task-scheduling-cron-atd.html


export EDITOR=vim
crontab -u username -e
创建一条每隔一分钟运行的命令
*/1 * * * * date >> /opt/test/log/cron.log


find命令的特殊用法
==================

查找/opt/temp 目录下的 名字扩展名为gz 修改时间在1分钟之前的文件 把找到的第1个匹配文件名作为 输入参数，调用/opt/collector.sh 来处理。  -quit表示找到第一个文件就退出。如果不指定-quit 则是对每个文件都进行处理
find /opt/temp -name '*.gz' -mmin +1  -execdir /opt/collector.sh {} \; -quit

删除所有一分钟之前的gz文件
find /opt/temp -name '*.gz' -mmin +1  -delete 

删除所有一天之前的gz文件
find /opt/temp -name '*.gz' -mtime +1  -delete 

显示所有最后修改在1分钟之前的目录
find /opt/temp -type d  -mmin +1

删除所有的空目录
find /opt/temp  -type d -empty -delete



