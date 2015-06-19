postgresql性能和tps相关的几个知识
=================================

1. 性能测试工具
----------------
可以执行指定的sql脚本文件，做压力测试，不过自己本身也有TPC-B 测试
apt-get install postgresql-contrib-9.4
[pgbench](http://www.postgresql.org/docs/9.4/static/pgbench.html)
pgbench -i -U smsc  smsc
先要初始化一下，它建几个用于测试的表，不过如果运行自定义sql脚本文件应该是用不上的
pgbench -r  -c 16 -T 30 -U smsc -f ./insert.sql smsc



2. 参考文档关于配置说明的部分，搞懂postgresql 数据库的每个参数
--------------------------------------------------------------
/etc/postgresql/9.3/main/postgresql.conf

## 基本概念:
### vacuum
      扫描磁盘数据，回收被delete或者update 导致变成 dead 的 block 等等，一般是由后台进程自己处理 auto vacuum
### checkpoint
      就是把insert update 导致的脏页写到磁盘。
### WAL write ahead log
      是 transaction 事务的log， 所有到事务操作都记录到这个log，保证数据的完整性。
      每个事务都会保证log完整的写到磁盘来才返回事务成功。

```
shared_buffers = 512MB        # 不需要太大，因为postgresql的不是使用direct io 还依赖系统的page cache来缓存的。
effective_cache_size  = 4GB   # 提示系统可以用来做page cache的内存大小，可以设置为系统内存的75%。只是提示数据库选择合适query plan

checkpoint_segments = 64      # 这两个控制checkpoint的频率和持续的时间。
checkpoint_timeout = 10min    # 跟上面一个类似，不过是时间限制，那个先到达都触发checkpoint操作
checkpoint_completion_target = 0.9   #  checkpoint的持续占总的比例， 可以避免checkpoint的突发io

fsync = on   # 强制等待wal log 写到磁盘才能保证数据的完整性，关闭之后数据完整性就不能保证了，不过性能更好。因为transaction的WAL就不刷数据到磁盘了。
synchronous_commit = on   # 默认强制保证所有的transaction 都等待磁盘操作完才会返回成功。如果关闭了，宕机后可能最最多3个wal_writer_delay 的时间窗口内的transaction的数据就丢失。 如果你能容许少量的事务在crash之后丢失，是可以考虑这个选项试试看。性能会好很多


vacuum_cost_delay = 10ms  #这个值是指vacuum在消耗多少资源后停顿多少时间,以便其他的操作可以使用更多的硬件资源.

log_min_duration_statement = 1000ms # 记录运行超过1秒的SQL到日志中, 一般用于跟踪哪些SQL执行时间长. /var/log/postgres/

log_checkpoints = on # 记录每一次checkpoint到日志中.

# 记录锁等待超过1秒的操作, 一般用于排查业务逻辑上的问题.
log_lock_waits = on
deadlock_timeout = 1s
```

其他还有很多参数也都可以自己测试了解一下的。



3. 避免lock 竞争
-----------------
同个表多个 insert，update是可以并行执行的。不过如果有复杂的trigger 或者逻辑的sql代码是，
可能存在比如update 表的同一行等等导致的锁等待。锁的阻塞等待会严重的影响多个client
连接的server进程的性能

- 症状
进程cpuu利用率很低，
ps -ef 的输出里可以看到命令行有 “update waiting” “insert waiting”时不时出现。

- 检查确认
数据库内部是有一个pg_locks 表可以看到所有 block的进程的锁的
参考这里的sql语句，
https://wiki.postgresql.org/wiki/Lock_dependency_information

解决死锁问题之后， 多个client connection 并行执行的insert 性能提升很明显可以达到1倍。



4. postgresql的client 连接的和进程
-----------------------------------
 postgresql 一个 client connection， server端是启动一个 server进程来服务的。

 连接数目的选择，如果 postgresql的 server进程是cpu消耗型，大多资源都
 消耗在运算上面，这时连接数目就不能设置的太多，根据cpu个数来选择，避免太多的
 context switch。 测试一下就会发现这种时候4核多线程的机器很少连接数目4个比32
 多个性能都要好很多的。  这种应该是典型的模式？？  所有数据都缓存在内存中，
 没有其他文件阻塞或者锁等待。

如果是大量transaction类型，大量的时间都是耗在transaction的WAL log的fsync磁盘
等待上面的话, cpu 利用率会显的非常低接近0。磁盘io利用率接近100%。
这时由于postgresql的 "group commit" 特性, 可以把上次磁盘flush操作时间内累积下来的
多个transaction的WAL 合并到一个磁盘操作里面来进行。这个会大大提到并行的transaction的
性能。 这种情况下一直增加 client connection 到200 ～ 300，都可能看到没秒完成
transaction数量在增加。后面会讲到。 简单的说对于这种大量事务，cpu又很低的情况，
连接数一直加的很多都可以看到性能提高。

根据网上文章，如果你物理硬盘限制的TPS是150 的话，通过不停增加连接数目的方式可以
最多得到1000 的TPS。 自己测试好像也是差不多的。


5.  TPS 限制和 IOPS
-------------------

数据库的一个特性就是保证 transaction的完整性。
对于每个insert ，update 等事务，数据库是这样保证数据的完整性的。
  1. 把这个操作转换记录成WAL record，可以恢复，复制等的log记录
  2. 更新 表 在内存里面的row所在的 page。 这个可以延迟写到磁盘。
    因为即使这个丢失了，也可以根据 1. 的wal来恢复。

  但是 第1步所产生的WAL必须保证完整。所以每个transaction都必须有 fsync 数据到
  磁盘，等待返回，确保transaction的WAL数据完整写入了物理硬盘。
  这个是一个随机写的磁盘io操作（可能因为同时也有其他第2步产生的脏数据块要写磁盘操作等？？）

  所以数据库每秒所能完成的事务量TPS （transaction per second) 严重依赖于磁盘的
  随机写操作random write input/output operation per second（IOPS）的性能。
  普通的机械硬盘，这个IOPS数据是很差的，用fio等工具测试随机读写性能，就可以发现
  我自己的测试机这个数值大概是200左右而已。 测试时用 "iostat -x 4"观察也可以
  看到请求在物理设备的service time 有4.5毫秒甚至更慢。
  这样对已一个client 连接， 数据库的TPS也就150 ～ 200 左右。非常合理的数据。
  由于这个原因，估计使用SSD硬盘TPS性能会大量提高，看网上数据，SSD的IOPS都是8000甚至几十万的。

  不过新版的postgresql是有个“group commit”的特性的，可以把多个transaction的
  WAL 记录放在一个磁盘操作里面进行。这样就可以大量提高并发的transaction的性能了。
  他默认会在一个transaction写磁盘的时间窗口里面利用一个队列缓存其他并行的server
  进程所产成的事务。然后等上次fsync/commit操作完整之后，就可以一次的累积下来的
  多个事务需要commit的数据一次性的写到磁盘。这个特性默认会起作用，所以如果你并行的
  事务越多，一次磁盘操作合并写入的transaction数据越多，最后看到TPS数据就越好。
  数据库配置的commit_delay 和 commit_siblings是和这个特性相关的。但不建议配置
  这两个参数了，因为group commit在默认的配置就可以很好工作了，commit_delay只会
  这特殊的场合才需要使用，可能造成没必要的延时。可以参考文档的说明。

  详细的解释可以参考列出的第二篇文章, 他的这个修改代码合并到postgresql之后，
  可以在大量的client connection的时候获得14000个TPS

  用pgbench 测试那就会发现  -c  参数越多，能完成TPC-B  测试TPS 就越多。



6.  批量插入数据 减少事务
-------------------------
因为tps数量是固定的，如果你想insert更多的内容要一个table。
那么可以考虑吧多个insert 放到一个transaction里面来进行。这样同样的TPS，你可以
完成更多的INSERT操作。
这个可以使用 client connection的设置 关闭auto commit，写了很多insertsql之后
才执行一次， 最后才commit一次。 也可以直接自己设置 begin transaction end
等等。 还有copy 命令，
根据别人使用例子，说是批量100-1000 records/commit 都是可以的。
可以自己测试得到最合适批量数目。



7. 其他工具和辅助模块, 命令等
------------------------------
- [pgfincore 可以控制哪个常用的表被加载到系统的缓存，避免磁盘请求](https://github.com/klando/pgfincore)
- 还有很多连接池的模块
- pg_stat_statement  统计所有sql执行的时间，列出最慢的到一个表里面
- explian analyze 在执行的sql直接加上这个，可以得到详细的性能统计， 索引使用的耗时等等
- create unlogged table    这种unlogged表可以不产生WAL log，
   当然也就不能保证完整性了，甚至每次数据库crash之后就清空这种表。但这种表的insert和
    update操作是非常快的。因为没有了transaction等待磁盘完成这一步。
- 表的fillfactor ， 有人说如果你表同时有insert 和update操作，这个可以设置成90等等。
  这样后面的update可能还有空间利用，可以把数据放在邻近的block或者page里面。但这样
  磁盘的空间占用更多。
- 表大小和 表一行占用空间的大小
   SELECT pg_size_pretty(pg_relation_size('tmp_tbl'));  -- complete size of table
   SELECT pg_column_size(t) FROM tmp_tbl t LIMIT 10;  -- size of sample rows
- 索引，也是是影响性能的。参考官方文档。并行减少锁竞争？
  CREATE INDEX CONCURRENTLY idx_salary ON employees(last_name, salary);
  CREATE INDEX CONCURRENTLY on tokens (substr(token), 0, 8)

7.  参考
---------
  - postgresql 的文档还是非常详细的，要仔细查看相关的东西，理解提到的相关概念。
  1. [Re: High Frequency Inserts to Postgres Database vs Writing to a File](http://www.postgresql.org/message-id/4AF18F92.2060007@2ndquadrant.com)
  2. [towards-14000-write-transactions-on-my] (http://pgeoghegan.blogspot.jp/2012/06/towards-14000-write-transactions-on-my.html)
