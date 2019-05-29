---
layout:       post
title:        "MySQL利用init-connect增加访问审计功能异常"
subtitle:     "init_connect"
header-img:   "img/MySQL.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - MySQL

---


#### init-connet设置
##### 注：该参数对超级用户不生效
```
-- 创建测试库
mysql> create database test;
Query OK, 1 row affected (0.00 sec)

mysql> use test;
Database changed

-- 创建审计记录表
mysql> CREATE TABLE `conn_log` (
    ->   `conn_id` int(11) DEFAULT NULL,
    ->   `conn_time` datetime DEFAULT NULL,
    ->   `user_name` varchar(128) CHARACTER SET utf8 DEFAULT NULL,
    ->   `cur_user_name` varchar(128) CHARACTER SET utf8 DEFAULT NULL,
    ->   `ip` varchar(15) CHARACTER SET utf8 DEFAULT NULL,
    ->   KEY `conn_time` (`conn_time`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 ;
Query OK, 0 rows affected (0.01 sec)

-- 设置审计内容
mysql> set global init_connect="set @user=user(),@cur_user=current_user();insert into test.conn_log values(connection_id(),now(),@user,@cur_user,'10.0.0.1');"
    -> ;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like '%init%';
+------------------------+-------------------------------------------------------------------------------------------------------------------------------+
| Variable_name          | Value                                                                                                                         |
+------------------------+-------------------------------------------------------------------------------------------------------------------------------+
| init_connect           | set @user=user(),@cur_user=current_user();insert into test.conn_log values(connection_id(),now(),@user,@cur_user,'10.0.0.1'); |
| init_file              |                                                                                                                               |
| init_slave             |                                                                                                                               |
| table_definition_cache | 1400                                                                                                                          |
+------------------------+-------------------------------------------------------------------------------------------------------------------------------+
4 rows in set (0.00 sec)

-- 创建普通用户

mysql> grant select,insert on dba_test.* to 'test'@'%' identified by 'test';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)

```


#### 异常
```
[root@test ~]# mysql -S /data0/mysql57/mysql3307/mysqltmp/mysql3307.sock  -utest -ptest 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 117
Server version: 5.7.21-log

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show user();
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    118
Current database: *** NONE ***

ERROR 1184 (08S01): Aborted connection 118 to db: 'unconnected' user: 'test' host: 'localhost' (init_connect command failed)
mysql> select user();
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    119
Current database: *** NONE ***


```

#### 异常处理
##### 分析
通过查看erro log发现test用户没有test.conn_log表的写权限，导致init-connect中的sql内容无法进行，从而导致连接失败


##### 解决
```
-- 赋权
mysql> grant insert on test.* to 'test'@'%';
Query OK, 0 rows affected (0.00 sec)
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

-- 登陆正常
[root@test ~]# mysql -hip地址 -P3307  -utest -ptest
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 140
Server version: 5.7.21-log MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use dba_test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+--------------------+
| Tables_in_dba_test |
+--------------------+
| user               |
+--------------------+
1 row in set (0.00 sec)

mysql> insert into user(user_id,username) values(4,'d');
Query OK, 1 row affected (0.00 sec)

mysql> 


```

