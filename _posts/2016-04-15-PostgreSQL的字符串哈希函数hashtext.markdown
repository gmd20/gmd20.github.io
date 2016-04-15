
psql 里面输入这个命令，可以显示所有名字带hash的函数
```text
postgres=# \df
                       List of functions
postgres=#  \df *hash*   Schema   |       Name        | Result data type |                         Argument data types                          |  Type  
------------+-------------------+------------------+----------------------------------------------------------------------+--------
 pg_catalog | hash_aclitem      | integer          | aclitem                                                              | normal
 pg_catalog | hash_array        | integer          | anyarray                                                             | normal
 pg_catalog | hash_numeric      | integer          | numeric                                                              | normal
 pg_catalog | hash_range        | integer          | anyrange                                                             | normal
 pg_catalog | hashbeginscan     | internal         | internal, internal, internal                                         | normal
 pg_catalog | hashbpchar        | integer          | character                                                            | normal
 pg_catalog | hashbuild         | internal         | internal, internal, internal                                         | normal
 pg_catalog | hashbuildempty    | void             | internal                                                             | normal
 pg_catalog | hashbulkdelete    | internal         | internal, internal, internal, internal                               | normal
 pg_catalog | hashchar          | integer          | "char"                                                               | normal
 pg_catalog | hashcostestimate  | void             | internal, internal, internal, internal, internal, internal, internal | normal
 pg_catalog | hashendscan       | void             | internal                                                             | normal
 pg_catalog | hashenum          | integer          | anyenum                                                              | normal
 pg_catalog | hashfloat4        | integer          | real                                                                 | normal
 pg_catalog | hashfloat8        | integer          | double precision                                                     | normal
 pg_catalog | hashgetbitmap     | bigint           | internal, internal                                                   | normal
 pg_catalog | hashgettuple      | boolean          | internal, internal                                                   | normal
 pg_catalog | hashinet          | integer          | inet                                                                 | normal
 pg_catalog | hashinsert        | boolean          | internal, internal, internal, internal, internal, internal           | normal
 pg_catalog | hashint2          | integer          | smallint                                                             | normal
 pg_catalog | hashint2vector    | integer          | int2vector                                                           | normal
 pg_catalog | hashint4          | integer          | integer                                                              | normal
 pg_catalog | hashint8          | integer          | bigint                                                               | normal
 pg_catalog | hashmacaddr       | integer          | macaddr                                                              | normal
 pg_catalog | hashmarkpos       | void             | internal                                                             | normal
 pg_catalog | hashname          | integer          | name                                                                 | normal
 pg_catalog | hashoid           | integer          | oid                                                                  | normal
 pg_catalog | hashoidvector     | integer          | oidvector                                                            | normal
 pg_catalog | hashoptions       | bytea            | text[], boolean                                                      | normal
 pg_catalog | hashrescan        | void             | internal, internal, internal, internal, internal                     | normal
 pg_catalog | hashrestrpos      | void             | internal                                                             | normal
 pg_catalog | hashtext          | integer          | text                                                                 | normal
 pg_catalog | hashvacuumcleanup | internal         | internal, internal                                                   | normal
 pg_catalog | hashvarlena       | integer          | internal                                                             | normal
 pg_catalog | interval_hash     | integer          | interval                                                             | normal
 pg_catalog | jsonb_hash        | integer          | jsonb                                                                | normal
 pg_catalog | pg_lsn_hash       | integer          | pg_lsn                                                               | normal
 pg_catalog | time_hash         | integer          | time without time zone                                               | normal
 pg_catalog | timestamp_hash    | integer          | timestamp without time zone                                          | normal
 pg_catalog | timetz_hash       | integer          | time with time zone                                                  | normal
 pg_catalog | uuid_hash         | integer          | uuid                                                                 | normal


```

字符串可以使用  hashtext  函数
源码对应的是
http://doxygen.postgresql.org/hashfunc_8c_source.html

看上对应的是 Jenkins hash算法

