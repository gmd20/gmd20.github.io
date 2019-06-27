# 下载最新的安装包

```text
rpm -i grafana-6.2.5-1.x86_64.rpm
rpm -i influxdb-1.7.6.x86_64.rpm
```

# influxdb 的配置

```text
对应的服务
service influxd start

配置文件
/etc/influxdb/influxdb.conf
默认http服务 端口为8086
数据和元数据的保存路径
dir = "/var/lib/influxdb/data"

influx>
 > use telegraf     选择要查看的数据库名
 > select * from table_name    查看数据
 > SHOW RETENTION POLICIES ON telegraf   查看数据保留策略配置
 > CREATE RETENTION POLICY "2_hours" ON "telegraf" DURATION 2h REPLICATION 1 DEFAULT  修改默认的保留的数据策略
 > drop retention POLICY "2_hours" ON "telegraf"   
 > CREATE RETENTION POLICY "a_year" ON "food_data" DURATION 52w REPLICATION 1
   CREATE CONTINUOUS QUERY "cq_30m" ON "food_data" BEGIN          设置 30分钟精度的平均值得保留一年吧。
      SELECT mean("website") AS "mean_website",mean("phone") AS "mean_phone"
      INTO "a_year"."downsampled_orders"
      FROM "orders"
      GROUP BY time(30m)                 
    END
  SELECT * FROM "orders" LIMIT 5
  SELECT * FROM "a_year"."downsampled_orders" LIMIT 5
```

# granfana的配置使用
```text
/bin/systemctl start grafana-server.service
vi /etc/grafana/grafana.ini
[server]默认设置0.0.0.0  3000
[security] 默认密码admin/admin

1. add data source，添加influxdb的 8086端口
2. 添加dashboard， 添加panel，添加query
3. dashboard 右上角选择时间段
```

# telegraf的配置里面配置把数据写到influxdb
```text
[[outputs.influxdb]]
  urls = ["http://192.168.1.4:8086"]                                            
  database = "test" 
```
