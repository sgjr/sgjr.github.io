
---
layout:       post
title:        "MySQL5.5升级至5.7"
subtitle:     "MySQL升级"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - MySQL
    - MySQL升级

---

### 一、准备工作
1. 新的服务器(10.12.21.184),作为从库
2. 在21.184上下载MySQL5.6、5.7的最新稳定版本的二进制包

host | role
---|---
10.12.21.120 | master
10.12.21.184 | slave




### 二、操作  
#### 1. 主从搭建
1. xtrbackup全备（20.120）
2. 根据全备在20.184上启动新的5.5数据库，作为20.120的从库
3. 启动主从，等待从库追上主库

#### 2. 升级从库       
1.解压文件包

```
cd /data0/mysql_update
tar xf mysql-5.6.40-linux-glibc2.12-x86_64.tar.gz
cd /usr/local
ln -s /data0/mysql_upgrade/mysql-5.6.40-linux-glibc2.12-x86_64 mysql
```
2.添加环境变量
```
export PATH=/usr/local/mysql/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/mysql/lib:$LD_LIBRARY_PATH
```


3.修改配置5.6文件
```
cd /data0/mysql/config/
cp my.cnf my56.cnf
#修改basedir,并注释该参数
# innodb_additional_mem_pool_size = 32M 从MySQL 5.6.3开始，   
#innodb_additional_mem_pool_size已被弃用，并将在未来的MySQL版本中删除。
#table_cache=8192
#thread_concurrency=48
basedir=/usr/local/mysql/
plugin_dir=/usr/local/mysql/lib/plugin


新增：
skip-slave-start
sql_mode=''

```


4.关闭快速关机参数
```
[root@localhost (none)]>set global innodb_fast_shutdown=0;
Query OK, 0 rows affected (0.00 sec)

[root@localhost (none)]>show variables like 'innodb_fast_shutdown';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| innodb_fast_shutdown | 0     |
+----------------------+-------+
1 row in set (0.00 sec)


```

5.关闭从库
```
[root@localhost local]# /data0/mysql/product/bin/mysqladmin --defaults-extra-file=/data0/mysql/config/user.root.cnf shutdown


```

6.启动
```
[root@localhost local]# /usr/local/mysql/bin/mysqld_safe --defaults-file=/data0/mysql/config/my56.cnf &
[1] 9446
[root@localhost local]# 180605 15:37:12 mysqld_safe Logging to '/data0/mysql/mysqllog/logfile/mysqld.err'.
180605 15:37:12 mysqld_safe Starting mysqld daemon with databases from /data0/mysql/dbdata

```

- 启动的时候会出现大量如下报错日志，不影响启动，属于正常现象
- 是因为MySQL5.5和5.6的表结构不同

```
2018-06-05 15:37:16 10312 [ERROR] Native table 'performance_schema'.'events_statements_summary_by_digest' has the wrong structure
2018-06-05 15:37:16 10312 [ERROR] Native table 'performance_schema'.'users' has the wrong structure
2018-06-05 15:37:16 10312 [ERROR] Native table 'performance_schema'.'accounts' has the wrong structure
2018-06-05 15:37:16 10312 [ERROR] Native table 'performance_schema'.'hosts' has the wrong structure
2018-06-05 15:37:16 10312 [ERROR] Native table 'performance_schema'.'socket_instances' has the wrong structure
2018-06-05 15:37:16 10312 [ERROR] Native table 'performance_schema'.'socket_summary_by_instance' has the wrong structure
2018-06-05 15:37:16 10312 [ERROR] Native table 'performance_schema'.'socket_summary_by_event_name' has the wrong structure
2018-06-05 15:37:16 10312 [ERROR] Native table 'performance_schema'.'session_connect_attrs' has the wrong structure
2018-06-05 15:37:16 10312 [ERROR] Native table 'performance_schema'.'session_account_connect_attrs' has the wrong structure


```



7.升级5.6
```
/usr/local/mysql/bin/mysql_upgrade -S /data0/mysql/dbdata/mysql.sock -uroot -pXXXXX
```

8.关闭5.6的库
```
[root@localhost local]# /usr/local/mysql/bin/mysqladmin
--defaults-extra-file=/data0/mysql/config/user.root.cnf shutdown

180605 16:02:30 mysqld_safe mysqld from pid file
/data0/mysql/dbdata/mysqld.pid ended
[1]+  Done                    /usr/local/mysql/bin/mysqld_safe --defaults-file=/data0/mysql/config/my56.cnf

```

9.解压MySQL5.7并建立软连接
```
cd /usr/local/
[root@localhost local]# ln -s /data0/mysql_upgrade/mysql-5.7.22-linux-glibc2.12-x86_64 mysql

```

10.修改配置文件，启动MySQL5.7
```
[root@localhost local]# /usr/local/mysql/bin/mysqld_safe --defaults-file=/data0/mysql/config/my57.cnf &
[1] 11262
[root@localhost local]# 2018-06-05T08:11:10.690409Z mysqld_safe Logging to '/data0/mysql/mysqllog/logfile/mysqld.err'.
2018-06-05T08:11:10.774997Z mysqld_safe Starting mysqld daemon with databases from /data0/mysql/dbdata


```

- 出现如下报错为正常现象
- 原因是5.6与5.7的系统表结构不同


```

018-06-05T08:11:14.340649Z 0 [ERROR] Native table 'performance_schema'.'events_transactions_current' has the wrong structure
2018-06-05T08:11:14.340681Z 0 [ERROR] Native table 'performance_schema'.'events_transactions_history' has the wrong structure
2018-06-05T08:11:14.340717Z 0 [ERROR] Native table 'performance_schema'.'events_transactions_history_long' has the wrong structure
2018-06-05T08:11:14.340774Z 0 [ERROR] Native table 'performance_schema'.'events_transactions_summary_by_thread_by_event_name' has the wrong structure
2018-06-05T08:11:14.340828Z 0 [ERROR] Native table 'performance_schema'.'events_transactions_summary_by_account_by_event_name' has the wrong structure
2018-06-05T08:11:14.340869Z 0 [ERROR] Native table 'performance_schema'.'events_transactions_summary_by_user_by_event_name' has the wrong structure
2018-06-05T08:11:14.340911Z 0 [ERROR] Native table 'performance_schema'.'events_transactions_summary_by_host_by_event_name' has the wrong structure
2018-06-05T08:11:14.340948Z 0 [ERROR] Native table 'performance_schema'.'events_transactions_summary_global_by_event_name' has the wrong structure
2018-06-05T08:11:14.345136Z 0 [ERROR] Incorrect definition of table performance_schema.users: expected column 'USER' at position 0 to have type char(32), found type char(16).

```

11.升级5.7(时间较长，放在后台)
```
 nohup /usr/local/mysql/bin/mysql_upgrade -S /data0/mysql/dbdata/mysql.sock -uroot -proot 2>&1 >/data0/mysql_upgrade/56to57.log &
```

```
www_fangdd_site.fdd_pats
Note     : TIME/TIMESTAMP/DATETIME columns of old format have been upgraded to the new format.
status   : OK

`cat`.`alteration`
Running  : ALTER TABLE `cat`.`alteration` FORCE
status   : OK
`cat`.`app_data_command_1`
Running  : ALTER TABLE `cat`.`app_data_command_1` FORCE
status   : OK



```


12.重启MySQL5.7数据库
```
[root@localhost mysql_upgrade]# /usr/local/mysql/bin/mysqladmin
--defaults-extra-file=/data0/mysql/config/user.root.cnf shutdown

[root@localhost mysql_upgrade]# /usr/local/mysql/bin/mysqld_safe --defaults-file=/data0/mysql/config/my57.cnf &


```


#### 三、升级后需要注意的问题
1.sql_mode
MySQL5.7默认为下列sql_mode，为避免以前语句不规范带来的隐患，将sql_mode设置为''
```
sql_mode                 | ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION

```
