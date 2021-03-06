---
title: "InfluxDB"
date: 2016-09-21 10:40
updated: 2016-09-21 10:40
logs: "新增"
---

[TOC]

InfluxDB是一个Go写的时序数据库(Time-Series Database)。

[官网](https://www.influxdata.com/time-series-platform/influxdb/) / [Github](https://github.com/influxdata/influxdb) / [文档](https://docs.influxdata.com/influxdb/latest/)


## 基本

环境：Gentoo

目前版本V1.0，没有提供官方的ebuild，不过`go-overlay`有别人写好的：<https://github.com/gentoo-mirror/go-overlay/tree/master/dev-db/influxdb>

也可以直接下载tar包，里面是Go编译好的，包括二进制工具，配置文件和目录等。

tar包里的配置有详细注释，默认的比如数据存储路径都是`/etc`下。

也可以手动生成配置。手动生成的配置默认存储路径等在$HOME/.influxdb/，且注释相对少一些：

```bash
$ cp ./etc/influxdb/influxdb.conf{,bak}
$ ./usr/bin/influxd config > ./etc/influxdb/influxdb.conf
```

启动influxdb：

```text
$ ./usr/bin/influxd run --config ./etc/influxdb/infludb.conf
```

默认会全局监听三个端口：

* 8083: Web admin管理服务的端口, <http://localhost:8083>
* 8086: HTTP API的端口
* 8088: 集群端口(目前还不是很清楚, 配置在全局的bind-address，默认不配置就是开启的)

官方给出的硬件配置是，[参考](https://docs.influxdata.com/influxdb/v1.0/introduction/installation#hardware)：

* 两块SSD。一块存储influxdb/wal，一块存储influxdb/data
* 最少8G内存

另外就是注意启动时的进程权限以及存储目录权限问题了。


### 核心概念

Influxdb是一个时序数据库，数据都是一些可度量的值，比如cpu load或者温度等。

写入是一行行的数据，称为`Point`：

```text
<measurement>[,<tag-key>=<tag-value>...] <field-key>=<field-value>[,<field2-key>=<field2-value>...] [unix-nano-timestamp]
```

由`time`(时间戳), `measurement`(比如cpuload), 至少一个k/v对`field`(measured value，比如value=0.6), 0个或多个k/v对`tags`(一些元信息metadata, 如host=server01, region=us)

官方给的例子：

```text
cpu,host=serverA,region=us_west value=0.64
payment,device=mobile,product=Notepad,method=credit billed=33,licenses=3i 1434067467100293230
stock,symbol=AAPL bid=127.46,ask=127.48
temperature,machine=unit42,type=assembly external=25,internal=37 1434067467000000000
```

和MySQL做类比，InfluxDB的`measurement`类似于表，主索引永远是time, tags是被索引的，fields不是；Point就是一行行的数据，只不过表名是在Point中。


### 常用操作

```text
# 创建数据库
> create database mydb

# 查看有哪些数据库, 在配置了认证的情况下，只有admin权限的用户可以看
> show databases;
name: databases
---------------
name
_internal  # <-- 这个是内置的，用于存储一些influxdb元信息
mydb

# 选择使用的数据库
> use mydb
Using database mydb

# 插入数据
> INSERT cpu,host=serverA,region=us_west value=0.62

# 查询数据
# 需要用 **双引号**
# 也可以 **都** 不用双引号
> SELECT "host", "region", "value" FROM "cpu"
name: cpu
---------
time                    host    region  value
1474442747173411091     serverA us_west 0.62

# 插入多个数据(fields)
> INSERT temperature,machine=unit42,type=assembly external=25,internal=37

# 除了glob匹配，InfluxQL还支持Go-Style的正则
> SELECT * FROM 'cpu'
ERR: error parsing query: found cpu, expected identifier at line 1, char 14
> SELECT * FROM "cpu"
name: cpu
---------
time                    host    region  value
1474442747173411091     serverA us_west 0.62

```


### 用户认证和权限

文档：[Authentication and Authorization](https://docs.influxdata.com/influxdb/v1.0/query_language/authentication_and_authorization/)

开启用户认证，首先创建admin权限用户和相应数据库的普通用户：

```text
# 默认没有用户
> show users;
user    admin  # 这里admin是一个boolean值

# 创建一个admin权限的用户
> create user admin with password '********' with all privileges
> show users;
user    admin
admin   true

# 创建一个普通权限的用户
> create user tankywoo with password '********'
> show users;
user       admin
admin      true
tankywoo   false

# 给普通用户赋予某个数据库的权限
> grant all on mydb to tankywoo

# 查看某个用户的权限
> show grants for tankywoo
database        privilege
mydb            ALL PRIVILEGES

# 删除用户
> drop user tankywoo
```

配置文件的`[http]`块：

```text
[http]
  enable = true
  bind-address = ":8086"
  auth-enabled = true  # 开启认证
  ...
```

重启influxd，下次使用influx cli时需要认证：

```text
# influx cli认证
# 然后进入交互输入用户名和密码
> auth
username: tankywoo
password:
```


## 术语

* InfluxQL: Influx Query Language


## 其它

* [influxdb-handbook](https://www.gitbook.com/book/xtutu/influxdb-handbook/details)
