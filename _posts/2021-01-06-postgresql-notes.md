---
title:  Postgresql 笔记
classes: wide
categories:
  - 2021-01
tags:
  - postgresql
type: note
---

Postgresql 学习使用中的一些技巧和心得，供以后参考。

## Backup & restore

pg_dump: [https://stackoverflow.com/questions/24718706/backup-restore-a-dockerized-postgresql-database](https://stackoverflow.com/questions/24718706/backup-restore-a-dockerized-postgresql-database)

```
docker exec -t oms-db pg_dumpall -c -U sa | gzip > ./dump_$(date +"%Y-%m-%d_%H_%M_%S").gz
gunzip < your_dump.sql.gz | docker exec -i oms-db psql -U sa -d ck
```

## Column alias

[https://www.enterprisedb.com/postgres-tutorials/how-use-tables-and-column-aliases-when-building-postgresql-querys](https://www.enterprisedb.com/postgres-tutorials/how-use-tables-and-column-aliases-when-building-postgresql-query)

- Column aliases can be used in the SELECT list of a SQL query in PostgreSQL.

- Like all objects, aliases will be in lowercase by default. If mixed-case letters or special symbols, or spaces are required, quotes must be used.

- Column aliases can be used for derived columns.

- Column aliases can be used with GROUP BY and ORDER BY clauses.

- We cannot use a column alias with WHERE and HAVING clauses.

**注意： 无法在 where 和 having 使用 column alias**

```
ck=# select id, wx_nickname || ' ' || real_name as name, case when length(real_name) > 0 then real_name else wx_nickname end as name2 from auth_user where name2 like '2%' order by id desc limit 30;
ERROR:  column "name2" does not exist
LINE 1: ...lse wx_nickname end as name2 from auth_user where name2 like..
```

原因及解决方案：

[https://dba.stackexchange.com/questions/225874/using-column-alias-in-a-where-clause-doesnt-work](https://dba.stackexchange.com/questions/225874/using-column-alias-in-a-where-clause-doesnt-work)

> The (historic) reason behind this is the sequence of events in a SELECT query. WHERE and HAVING are resolved before column aliases are considered, while GROUP BY and ORDER BY happen later, after column aliases have been applied.

使用sub query https://www.postgresqltutorial.com/postgresql-subquery/

```
ck=# select id, name2 from (select id, wx_nickname || ' ' || real_name as name, case when length(real_name) > 0 then real_name else wx_nickname end as name2 from auth_user order by id desc) as t where name2 like '2%';
 id | name2 
----+-------
 15 | 22
  1 | 222
(2 rows)
```

## Explain analyze

https://thoughtbot.com/blog/reading-an-explain-analyze-query-plan

## Time series data model (时序数据)

https://medium.com/@neslinesli93/how-to-efficiently-store-and-query-time-series-data-90313ff0ec20

https://medium.com/@valyala/measuring-vertical-scalability-for-time-series-databases-in-google-cloud-92550d78d8ae

## Graph data model（图数据）

https://news.ycombinator.com/item?id=10316872

## WITH Queries (Common Table Expressions CTE)

https://www.postgresql.org/docs/current/queries-with.html

[https://www.postgresqltutorial.com/postgresql-subquery/](https://www.postgresqltutorial.com/postgresql-subquery/)

主要用来查询层级数据，例如返回id为2的用户的所有下属

```
CREATE TABLE employees (
	employee_id serial PRIMARY KEY,
	full_name VARCHAR NOT NULL,
	manager_id INT
);

WITH RECURSIVE subordinates AS (
	SELECT
		employee_id,
		manager_id,
		full_name
	FROM
		employees
	WHERE
		employee_id = 2
	UNION
		SELECT
			e.employee_id,
			e.manager_id,
			e.full_name
		FROM
			employees e
		INNER JOIN subordinates s ON s.employee_id = e.manager_id
) SELECT
	*
FROM
    subordinates;
```

## JSONB column

[https://rollout.io/blog/unleash-the-power-of-storing-json-in-postgres/](https://rollout.io/blog/unleash-the-power-of-storing-json-in-postgres/)

关系型数据库为什么要存储 json 数据？

1. Avoid complicated joins on data that is siloed or isolated. 避免对孤立的数据进行无意义的join操作。

2. Avoid transforming data before returning it via your JSON API. json 数据无需处理可以直接返回。

例如: 供应商有多种工艺属性，两种实现方案：
 
- 可以选择新建工艺表与供应商表关联;

- 也可以新增 data 字段，类型为 jsonb, {"process": ["机加工", "版金"]}


## 按照中文拼音排序

https://github.com/digoal/blog/blob/master/201612/20161205_01.md

## select count(*)

数据量在 100w 级别的表上，执行该语句需要 100多毫秒，因为需要全表扫描。

https://www.cybertec-postgresql.com/en/postgresql-count-made-fast/

优化方案：

如果业务上对这个数字的实时性要求不高，可以在应用这边做缓存，1小时过期。


## unnest (list join table)

```
SELECT t.*
FROM unnest(ARRAY[1,2,3,2,3,5]) item_id
LEFT JOIN items t on t.id=item_id
```

https://stackoverflow.com/questions/2486725/postgresql-join-with-array-type-with-array-elements-order-how-to-implement

## Listen/Notify & Trigger

https://blog.lelonek.me/listen-and-notify-postgresql-commands-in-elixir-187c49597851

postgres 支持异步pub/sub，客户端可以收到数据库的消息通知。

**使用场景：**

- 数据表实时备份，对应用透明。

- 不修改A应用逻辑下，B应用能实时同步到A应用的数据。

**缺点**

数据库连接中断后，需要重新监听，存在数据丢失的情况。

解决方案：使用触发器把数据写入新的临时表，再异步消费。参考：https://billtian.github.io/digoal.blog/2018/07/13/03.html


实现步骤:

### 1. 创建数据库函数，发送通知到 test channel

```
    CREATE OR REPLACE FUNCTION notify_op_log_changes()
    RETURNS trigger AS $$
    BEGIN
    PERFORM pg_notify(
    'test',
    json_build_object(
    'operation', TG_OP,
    'record', row_to_json(NEW)
    )::text
    );

    RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;
```

### 2. 创建触发器，数据插入或更新后执行上面的函数

```
AFTER INSERT OR UPDATE
ON cybase_op_log
FOR EACH ROW
EXECUTE PROCEDURE notify_op_log_changes();
```

### 3. 测试脚本，订阅 test channel，接收消息

```
conn = psycopg2.connect(...)
conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)

curs = conn.cursor()
curs.execute("LISTEN test;")

print("Waiting for notifications on channel 'test'")
while True:
    if select.select([conn],[],[],5) == ([],[],[]):
        print("Timeout")
    else:
        conn.poll()
        while conn.notifies:
            notify = conn.notifies.pop(0)
            print("Got NOTIFY:", notify.pid, notify.channel, notify.payload)
```
