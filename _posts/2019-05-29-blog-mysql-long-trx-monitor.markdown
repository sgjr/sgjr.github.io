---
layout:       post
title:        "监控MySQL长事务并进行打点报警"
subtitle:     ""
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - MySQL

---


> 项目地址：https://github.com/sgjr/mysql_long_trx_monitor


此脚本主要用来监控MySQL主库的长事务， 通过读取mysql.cfg配置文件，获取MySQL连接地址，以及告警阈值 日志按天记录至当前文件夹下边的log目录中，日志格式为YYYY-mm-dd.log 通过pushgateway进行打点

## 准备工作
需要先部署好以下工具

- prometheus
- grafana
- pushgateway

安装好python依赖，修改脚本相关参数即可。

## 监控效果
![](https://github.com/sgjr/sgjr.github.io/blob/master/_posts/image/monitor_view_20190529.png)
## 报警效果
![](https://github.com/sgjr/sgjr.github.io/blob/master/_posts/image/alert_view_20190529.png)


