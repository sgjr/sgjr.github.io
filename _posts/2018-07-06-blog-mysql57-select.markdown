---
layout:       post
title:        "MySQL5.7查询性能改进"
subtitle:     "select in 5.7"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - MySQL
    - MySQL5.7

---

### 1.子查询
##### 1.1 MySQL5.5
```
mysql> explain extended select id,k,c,pad from sbtest1 where id in (select id from sbtest1 where k in ('50385','50011','43490','504922'));
+----+--------------------+---------+-----------------+---------------+---------+---------+------+--------+----------+-------------+
| id | select_type        | table   | type            | possible_keys | key     | key_len | ref  | rows   | filtered | Extra       |
+----+--------------------+---------+-----------------+---------------+---------+---------+------+--------+----------+-------------+
|  1 | PRIMARY            | sbtest1 | ALL             | NULL          | NULL    | NULL    | NULL | 612555 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | sbtest1 | unique_subquery | PRIMARY,k_1   | PRIMARY | 4       | func |      1 |   100.00 | Using where |
+----+--------------------+---------+-----------------+---------------+---------+---------+------+--------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```
##### 1.2 MySQL5.7
```
mysql> explain select id,k,c,pad from sbtest1 where id in (select id from sbtest1 where k in ('50385','50011','43490','50492'));
+----+-------------+---------+------------+--------+---------------+---------+---------+-------------------+------+----------+--------------------------+
| id | select_type | table   | partitions | type   | possible_keys | key     | key_len | ref               | rows | filtered | Extra                    |
+----+-------------+---------+------------+--------+---------------+---------+---------+-------------------+------+----------+--------------------------+
|  1 | SIMPLE      | sbtest1 | NULL       | range  | PRIMARY,k_1   | k_1     | 4       | NULL              |  253 |   100.00 | Using where; Using index |
|  1 | SIMPLE      | sbtest1 | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | sbtest.sbtest1.id |    1 |   100.00 | NULL                     |
+----+-------------+---------+------------+--------+---------------+---------+---------+-------------------+------+----------+--------------------------+
2 rows in set, 1 warning (0.00 sec)


```

### 2.union all
##### 2.1 MySQL5.5,会将结果存在临时表中
```
mysql> explain (select k from sbtest1 order by k) union all (select k from sbtest2 order by k);
+----+--------------+------------+-------+---------------+------+---------+------+--------+-------------+
| id | select_type  | table      | type  | possible_keys | key  | key_len | ref  | rows   | Extra       |
+----+--------------+------------+-------+---------------+------+---------+------+--------+-------------+
|  1 | PRIMARY      | sbtest1    | index | NULL          | k_1  | 4       | NULL | 612555 | Using index |
|  2 | UNION        | sbtest2    | index | NULL          | k_2  | 4       | NULL | 615365 | Using index |
| NULL | UNION RESULT | <union1,2> | ALL   | NULL          | NULL | NULL    | NULL |   NULL |             |
+----+--------------+------------+-------+---------------+------+---------+------+--------+-------------+
3 rows in set (0.00 sec)


```

##### 2.2 MySQL5.7,直接展示结果
```
mysql> explain (select k from sbtest1 order by k) union all (select k from sbtest2 order by k);
+----+-------------+---------+------------+-------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+------+---------+------+--------+----------+-------------+
|  1 | PRIMARY     | sbtest1 | NULL       | index | NULL          | k_1  | 4       | NULL | 597600 |   100.00 | Using index |
|  2 | UNION       | sbtest2 | NULL       | index | NULL          | k_2  | 4       | NULL | 597744 |   100.00 | Using index |
+----+-------------+---------+------------+-------+---------------+------+---------+------+--------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)


```

### 3 in查询
##### 3.1 MySQL5.5
```
mysql> explain select * from sbtest1 where (k,pad) in ((43490,'24909597713-10795827686-60824686337-78820064088-50914299985'),(50088,'56702105543-74313438035-88959810983-96828764563-29757615888'));
+----+-------------+---------+------+---------------+------+---------+------+--------+-------------+
| id | select_type | table   | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+----+-------------+---------+------+---------------+------+---------+------+--------+-------------+
|  1 | SIMPLE      | sbtest1 | ALL  | NULL          | NULL | NULL    | NULL | 612555 | Using where |
+----+-------------+---------+------+---------------+------+---------+------+--------+-------------+
1 row in set (0.00 sec)

```

##### 3.2 MySQL5.7
```
mysql> explain select * from sbtest1 where (k,pad) in ((43490,'24909597713-10795827686-60824686337-78820064088-50914299985'),(50088,'56702105543-74313438035-88959810983-96828764563-29757615888'));
+----+-------------+---------+------------+-------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | sbtest1 | NULL       | range | k_1           | k_1  | 4       | NULL |   77 |    20.00 | Using where |
+----+-------------+---------+------------+-------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

```
