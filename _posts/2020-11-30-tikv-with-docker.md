---
title:  "使用 docker-compose 启动 tikv 测试示例"
author: adream307
date:   2020-11-30 20:22:00 +0800
categories: [linux,tikv]
tags: [tikv]
---

`docker-compose.yml`
```yml
version: '3.5'

services:
 pd0:
   image: pingcap/pd:latest
   network_mode: "host"
   ports:
     - "2379:2379"
     - "2380:2380"
   volumes:
     - /etc/localtime:/etc/localtime:ro
   command:
     - --name=pd0
     - --client-urls=http://0.0.0.0:2379
     - --peer-urls=http://0.0.0.0:2380
     - --advertise-client-urls=http://127.0.0.1:2379
     - --advertise-peer-urls=http://127.0.0.1:2380
     - --initial-cluster=pd0=http://127.0.0.1:2380
     - --data-dir=/data/pd0
     - --log-file=/logs/pd0.log

 tikv0:
   network_mode: "host"
   image: pingcap/tikv:latest
   ports:
     - "20160:20160"
   volumes:
     - /etc/localtime:/etc/localtime:ro
   command:
     - --addr=0.0.0.0:20160
     - --advertise-addr=127.0.0.1:20160
     - --data-dir=/data/tikv0
     - --pd=127.0.0.1:2379
     - --log-file=/logs/tikv0.log
   depends_on:
     - "pd0"
```
注意， `network_mode` 为`host`

启动 `tidb-server` 使用 `tikv`

```bash
 ${TIDB}/bin/tidb-server --store=tikv --path="127.0.0.1:2379"
```

使用 `mysql` 客户端链接 `tidb-server`

```bash
$ mysql -h 127.0.0.1 -P 4000 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.25-TiDB-v4.0.0-beta.2-1667-gc9288d246 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use test;
Database changed
mysql> create table t1(a int);
Query OK, 0 rows affected (0.14 sec)

mysql> insert into t1 values(0);
Query OK, 1 row affected (0.03 sec)

mysql> insert into t1 values(1);
Query OK, 1 row affected (0.02 sec)

mysql> insert into t1 values(2);
Query OK, 1 row affected (0.02 sec)

mysql> select * from t1;
+------+
| a    |
+------+
|    0 |
|    1 |
|    2 |
+------+
3 rows in set (0.00 sec)
```
