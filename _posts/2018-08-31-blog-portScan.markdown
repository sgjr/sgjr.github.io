---
layout:       post
title:        "使用python-nmap模块扫描端口脚本"
subtitle:     "python"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - script

---


# portScan
端口扫描

项目地址：https://github.com/sgjr/portScan

### 使用须知
使用前需要安装nmap命令以及python-nmap模块
```
yum install nmap
pip install python-nmap
```

### 使用方法
```
Usage: portScan.py [Options]

Options:
  --version             show program's version number and exit
  -h, --help            show this help message and exit
  -H SCANHOST, --host=SCANHOST
                        The hosts will be scan
  -p SCANPORT, --port=SCANPORT
                        The ports will be scan
```

### 使用示例
1.python portScan.py -H 127.0.0.1 -p 3306

```
 ----- 127.0.0.1 -----
127.0.0.1 tcp/3306 open

```

2.python portScan.py -H 127.0.0.1 -p 3306-3308
```
 ----- 127.0.0.1 -----
127.0.0.1 tcp/3306 open
127.0.0.1 tcp/3307 open
127.0.0.1 tcp/3308 closed
```
3.python portScan.py -H 127.0.0.1 -p 3306,3309
```
 ----- 127.0.0.1 -----
127.0.0.1 tcp/3306 open
127.0.0.1 tcp/3309 closed

```
4.python portScan.py -H 127.0.0.1-2 -p 3306-3308
```
 ----- 127.0.0.1 -----
127.0.0.1 tcp/3306 open
127.0.0.1 tcp/3307 open
127.0.0.1 tcp/3308 closed

 ----- 127.0.0.2 -----
127.0.0.2 tcp/3306 open
127.0.0.2 tcp/3307 open
127.0.0.2 tcp/3308 closed

```
