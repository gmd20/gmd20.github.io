参考 https://docs.influxdata.com/influxdb/v1.7/query_language/

```text
influx -ssl -unsafeSsl   # 连接数据库
influx -ssl -unsafeSsl  -precision rfc3339 # 指定默认时间格式
> auth  # 用户名认证
> show databases  # 查看数据库
> use <database-name> # 选择数据库
> show measurements   # 查看监控指标
> select * from <measurement> where <tag|field> = 'xxx' and time > '2019-12-18T00:00:00Z' and time < '2019-12-18T00:01:00Z'
> precision rfc3339  # 时间格式切换
> select (a - b - c)*100/(a + 1) from <measurement> where <tag|field> = 'xxx' and time > '2019-12-17T12:00:00Z' and time < '2019-12-19T00:00:00Z' tz('Asia/Shanghai')  # 时区和基本运算

```
