---
layout:       post
title:        "MySQL复制过滤的坑"
subtitle:     "replicate-ignore-db replicate-do-db"
header-img:   "img/mysql.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - MySQL
    - 数据库参数
    
---

### 介绍
MySQL5.7官方文档关于相关参数的介绍：https://dev.mysql.com/doc/refman/5.7/en/change-replication-filter.html

在5.7版本支持动态修改，之前的版本需要重启数据库：
```
STOP SLAVE SQL_THREAD;

CHANGE REPLICATION FILTER REPLICATE_IGNORE_DB=(demo);

START SLAVE SQL_THREAD;

```

### 问题描述
在主库执行语句，如果不使用use db;  
DDL语句不会在从库执行，进而导致主从错误

### 场景复现

从库配置：
```

replicate-ignore-db = test
replicate-do-db = abc

```

主库操作：
```
22:29:04test>use test;
Database changed
22:30:26test>create table abc.t0417(id int,name varchar(20));
Query OK, 0 rows affected (0.08 sec)

22:31:37abc>use test;
Database changed
22:32:05test>insert into abc.t0417 values(1,'a');
Query OK, 1 row affected (0.02 sec)


```

从库状态,SQL线程出现报错：
```
22:31:07abc>show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.136.128
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000015
          Read_Master_Log_Pos: 8478
               Relay_Log_File: cptest-relay-bin.000007
                Relay_Log_Pos: 1292
        Relay_Master_Log_File: mysql-bin.000015
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: abc
          Replicate_Ignore_DB: test
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1146
                   Last_Error: Error executing row event: 'Table 'abc.t0417' doesn't exist'
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 8235
              Relay_Log_Space: 1709
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 1146
               Last_SQL_Error: Error executing row event: 'Table 'abc.t0417' doesn't exist'
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 4ac80f6d-8063-11e7-9d63-000c291d913c
             Master_Info_File: /data/mysql/mysql3309/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 180416 22:32:20
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 4ac80f6d-8063-11e7-9d63-000c291d913c:3609-3630
            Executed_Gtid_Set: 4ac80f6d-8063-11e7-9d63-000c291d913c:1:3609-3629,
a1c747e9-4170-11e8-883f-000c29c6b279:1
                Auto_Position: 0
1 row in set (0.00 sec)




```


#### 解决问题
更改从库配置：
```
#replicate-ignore-db = test
#replicate-do-db = abc
replicate_wild_do_table = abc.%
replicate_wild_ignore_table = test.%


```


主库操作:

```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 65
Server version: 5.6.35-log Source distribution

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

caopeng@192.168.136.128 00:39:16(none)>use test;
Database changed
caopeng@192.168.136.128 00:39:32test>create table abc.test0417(id int,name varchar(20));
Query OK, 0 rows affected (0.17 sec)

caopeng@192.168.136.128 00:40:25test>insert into abc.test0417 values(1,'b');
Query OK, 1 row affected (0.11 sec)



```


从库状态：
```
root@localhost 00:39:06(none)>show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.136.128
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000015
          Read_Master_Log_Pos: 9644
               Relay_Log_File: cptest-relay-bin.000010
                Relay_Log_Pos: 771
        Relay_Master_Log_File: mysql-bin.000015
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: abc.%
  Replicate_Wild_Ignore_Table: test.%
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 9644
              Relay_Log_Space: 945
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 4ac80f6d-8063-11e7-9d63-000c291d913c
             Master_Info_File: /data/mysql/mysql3309/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 4ac80f6d-8063-11e7-9d63-000c291d913c:3609-3635
            Executed_Gtid_Set: 4ac80f6d-8063-11e7-9d63-000c291d913c:1:3609-3635,
a1c747e9-4170-11e8-883f-000c29c6b279:1-6
                Auto_Position: 0
1 row in set (0.00 sec)

root@localhost 00:40:54(none)>select * from abc.test0417;
+------+------+
| id   | name |
+------+------+
|    1 | b    |
+------+------+
1 row in set (0.00 sec)



```

#### 总结
- 使**用replicate_do_db**和**replicate_ignore_db**两个参数时在主库操作需要使用use db;然后再进行其他操作
- 推荐使用**replicate_wild_do_table，
replicate_wild_ignore_table**


