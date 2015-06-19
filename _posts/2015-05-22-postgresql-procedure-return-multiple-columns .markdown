---
title:  "Postgresql的存储过程返回表的多个column(row)的例子"
---

postgresql支持的存储过程支持标准的sql语法和自己扩展的pl/pgsql。甚至c的代码，不知道那个怎么写了。

Stored procedure的优点
--------------------------------
http://en.wikipedia.org/wiki/Stored_procedure


标准sql
----------

```sql
CREATE OR REPLACE FUNCTION test_select(IN id text) RETURNS SETOF test_table_name AS $$
SELECT * FROM test_talbe_name where test_talbe_name.text = id;
$$ LANGUAGE sql STABLE;

ALTER FUNCTION test_select(text) OWNER TO test_user;
SELECT * FROM test_select('3');
```



- postgresql 的pl/pgsql
-----------------------

http://www.postgresql.org/docs/9.4/static/plpgsql.html

```sql
CREATE OR REPLACE FUNCTION test_select(IN id text) RETURNS test_talbe_name AS $$
DECLARE
    sms_row test_talbe_name%ROWTYPE;
BEGIN
SELECT * into sms_row FROM test_talbe_name where test_talbe_name.text = id;
RETURN sms_row;
END;
$$ LANGUAGE plpgsql STABLE;

ALTER FUNCTION test_select(text) OWNER TO test_user;

SELECT * FROM test_select('3');
s
```


```sql
CREATE OR REPLACE FUNCTION select_smpp_sms(IN sms_id text) RETURNS TABLE (
  id text,
  session_id integer,
  text text,
 ) AS $$
BEGIN
    return query
    SELECT id,session_id,text
    FROM ONLY sms WHERE sms.id = sms_id;
END;
$$ LANGUAGE plpgsql STABLE;
```
