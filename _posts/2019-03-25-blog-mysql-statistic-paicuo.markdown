---
layout:       post
title:        "MySQL统计信息异常"
subtitle:     "innodb_stats_on_metadata"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - MySQL

---

### 起因
在MySQL服务器运行mysqld_exporter后，发现数据库的中活跃连接数暴增，而且都是来自于mysqld_exporter的慢查询，语句如下：
```
SELECT
            TABLE_SCHEMA,
            TABLE_NAME,
            TABLE_TYPE,
            ifnull(ENGINE, 'NONE') as ENGINE,
            ifnull(VERSION, '0') as VERSION,
            ifnull(ROW_FORMAT, 'NONE') as ROW_FORMAT,
            ifnull(TABLE_ROWS, '0') as TABLE_ROWS,
            ifnull(DATA_LENGTH, '0') as DATA_LENGTH,
            ifnull(INDEX_LENGTH, '0') as INDEX_LENGTH,
            ifnull(DATA_FREE, '0') as DATA_FREE,
            ifnull(CREATE_OPTIONS, 'NONE') as CREATE_OPTIONS
          FROM information_schema.tables
          WHERE TABLE_SCHEMA = 'xxx';

```


### 分析
1.在该数据库执行该语句，执行时间非常慢
```
102 rows in set (6.35 sec)

```

2.在该数据库的从库执行，结果却完全不一样
```
102 rows in set (0.01 sec)
```

3.这个时候就可以确定应该跟MySQL统计信息有关。

> 查看MySQL统计信息相关介绍：https://blog.csdn.net/n88Lpo/article/details/79144495

4.查看主从数据库的参数，发现差异
```
##主库
mysql> show variables like 'innodb_stats_on_metadata';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_stats_on_metadata | ON    |
+--------------------------+-------+
1 row in set (0.00 sec)


##从库
mysql> show variables like 'innodb_stats_on_metadata';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_stats_on_metadata | OFF   |
+--------------------------+-------+
1 row in set (0.00 sec)

```

5.确定原因为每次查询时都会对统计信息进行更新。

> 查看MySQL官方文档 https://dev.mysql.com/doc/refman/5.7/en/innodb-statistics-estimation.html

非持久化统计信息在以下情况会被自动更新|
---|
1 执行ANALYZE TABLE| 
2 innodb_stats_on_metadata=ON情况下，执SHOW TABLE STATUS, SHOW INDEX, 查询 INFORMATION_SCHEMA下的TABLES, STATISTICS |
3 启用--auto-rehash功能情况下，使用mysql client登录 |
4 表第一次被打开 |
5 距上一次更新统计信息，表1/16的数据被修改 |


### 解决
修改参数`innodb_stats_on_metadata`
```
mysql> set global innodb_stats_on_metadata=0;
Query OK, 0 rows affected (0.00 sec)

```


