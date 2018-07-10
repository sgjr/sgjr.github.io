---
layout:       post
title:        "python虚拟环境"
subtitle:     "virtualenv"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - python

---

1、**安装virtualenv**
```
pip3 install virtualenv
```

2、**使用virtualenv命令创建虚拟环境**
```
virtualenv --no-site-packages -p /usr/local/python3/bin/python3 venv
           没有类包的干净环境       指定python版本           环境地址

```

3、**进入虚拟环境**
```

source venv/bin/activate
```

4、**在虚拟环境中完全与系统隔离的python环境**，可以在里边通过pip安装类包
```
pip install django==1.11.6
pip install uwsgi

```

5、**使用虚拟环境运行项目**
```
/.../venv/bin/python manage.py runserver 0.0.0.0:8000

```

6、**退出virtualenv**
```
deactivate

```
