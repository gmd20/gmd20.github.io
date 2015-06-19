---
title:  "posgresql不明数据文件占用大量磁盘空间"
---

对某应用做压力测试，发现postgresql的把磁盘空间用完了，数据库操作都失败了。
-------------------------------------------------
因为这个应用有对某表的大量insert/update/delete（移到历史表)的操作。
怀疑是历史表数据太多了，在pgadmin 里面操作删除了历史表，结果磁盘空间没有明显改变。
对数据库和表做 VACUUM full 操作，磁盘占用不变。

用下面的数据库命令查看，确实是改测试数据库占用了磁盘空间将近50GB，但改数据库里面
最大的表占用的空间才400多MB。不清楚磁盘占用到底用在哪里了。


查看数据库和表占用的磁盘空间
----------------------------
[Disk Usage](https://wiki.postgresql.org/wiki/Disk_Usage)
[28.1. Determining Disk Usage](http://www.postgresql.org/docs/9.4/static/disk-usage.html)
[9.26. System Administration Functions] (http://www.postgresql.org/docs/9.4/static/functions-admin.html)



- 最大的数据库有47GB
```sql
SELECT d.datname AS Name,  pg_catalog.pg_get_userbyid(d.datdba) AS Owner,
    CASE WHEN pg_catalog.has_database_privilege(d.datname, 'CONNECT')
        THEN pg_catalog.pg_size_pretty(pg_catalog.pg_database_size(d.datname))
        ELSE 'No Access'
    END AS Size
FROM pg_catalog.pg_database d
    ORDER BY
    CASE WHEN pg_catalog.has_database_privilege(d.datname, 'CONNECT')
        THEN pg_catalog.pg_database_size(d.datname)
        ELSE NULL
    END DESC -- nulls first
    LIMIT 20;
```


- 最大的表才 400多MB
```sql
SELECT nspname || '.' || relname AS "relation",
    pg_size_pretty(pg_total_relation_size(C.oid)) AS "total_size"
  FROM pg_class C
  LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
  WHERE nspname NOT IN ('pg_catalog', 'information_schema')
    AND C.relkind <> 'i'
    AND nspname !~ '^pg_toast'
  ORDER BY pg_total_relation_size(C.oid) DESC
  LIMIT 20;
```





有网友跟我说可能是pg_xlog， 也就是WAL (Write Ahead Log) files
------------------------
看一下这个目录，其实很小，不是这个的原因

root@TEST-LINUX-07:/opt/# du -sh /var/lib/postgresql/9.3/main/pg_xlog/
1.1G	/var/lib/postgresql/9.3/main/pg_xlog/


确实是数据库的文件
------------------
有很多1G大小的文件

```
root@TEST-LINUX-07:/opt/#  du -sh /var/lib/postgresql/9.3/main/base/16385/*
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.1
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.10
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.11
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.12
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.13
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.14
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.15
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.16
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.17
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.18
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.19
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.2
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.20
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.21
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.22
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.23
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.24
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.25
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.26
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.27
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.28
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.29
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.3
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.30
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.31
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.32
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.33
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.34
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.35
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.36
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.37
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.38
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.39
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.4
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.40
875M	/var/lib/postgresql/9.3/main/base/16385/20401.41
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.5
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.6
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.7
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.8
1.1G	/var/lib/postgresql/9.3/main/base/16385/20401.9
1.1G	/var/lib/postgresql/9.3/main/base/16385/20413
1.1G	/var/lib/postgresql/9.3/main/base/16385/20413.1
1.1G	/var/lib/postgresql/9.3/main/base/16385/20413.2
1.1G	/var/lib/postgresql/9.3/main/base/16385/20413.3
```


postgresql数据文件的结构
------------------------
[58.1. Database File Layout](http://www.postgresql.org/docs/9.3/static/storage-file-layout.html)

参考文档，肯定是数据库对应的文件占用了空间了，
/var/lib/postgresql/9.3/main/base/16385/20401.38
16385 是对应某个数据库的oid，20401 对应的是某个表或者索引的relfilenode
pgadmin 查看该测试用的数据库oid确实是 16385 没错

用sql 命令查看数据库的oid
```sql
SELECT oid,* from pg_database
```

查看某个表的oid
```sql
SELECT oid,relname,relfilenode FROM pg_class order by oid;
```
没有找到20401  对应的是什么来的。

relfilenode  对应是磁盘文件名，就是pg_relation_filenode/pg_relation_filepath 函数的输出
使用表的oid 查看某个表的在磁盘的文件
```
select from pg_relation_filepath(16897);
select pg_relation_filepath('table_name/index_nam');
```

根据文档，postgresql确实是按照1GB来分割文件的。  但在pg_class 里面找不到那个
表或者其他索引之类的类型的 relfilenode 是 对应20401 这些占用磁盘空间的数据文件的。



孤儿文件(orphaned large objects)
--------------------------------
[F.46. vacuumlo] (http://www.postgresql.org/docs/9.3/static/vacuumlo.html)
[F.20. lo](http://www.postgresql.org/docs/9.3/static/lo.html)

以前操作没有完整的删除数据文件，查看我上面的文件确实内容有点像我已经在pgadmin里面
删除的一个“继承的历史表，子表文件”。 由于某些原因（连接断开？）pgadmin没有把改
table相关的数据文件完整删除？

vacuumlo -n <database_name> -U <username> -W  -h localhost -p 5432


"Large objects"  是一个postgresql特性来的，跟 base目录下的文件无关



用oid2name查看oid是什么东西来的
----------------------
oid2name 可以根据oid 或者filenode 反过来查找对应的table 名字
 /usr/lib/postgresql/9.3/bin/oid2name  -d snsc -f 20401 -H localhost -p 5432 -U <username>
Password:
From database "smsc":
  Filenode  Table Name
----------------------

就是查看不20401的对应表名字是什么。该目录下的其他文件都是可以查看的出来的。


* 还有一些有趣的工具

- /usr/lib/postgresql/9.3/bin/pg_dump 把数据库备份为sql

- pg_xlogdump display a human-readable rendering of the write-ahead log of a PostgreSQL database cluster.

- vacuumdb -- garbage-collect and analyze a PostgreSQL database 对vacuumd 命令的一个简单的包装而已。


最后的结果
----------
vacuumdb ，重启数据库，这些物理文件也不会消失。
尝试用pg_dump 备份一个整个数据库，备份也是很小的。
看来只能这些物理文件是数据库没能正常删除的无用遗留文件来的。很功能就是我删除那个比较大的历史表的时候遗留下来的。
自己手工把他删除算了。
