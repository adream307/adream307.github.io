---
title:  "使用 Docker 启动 mongodb"
author: adream307
date:   2020-10-20 18:30:00 +0800
categories: [mongo, docker]
tags: [docker, mongodb]
---

## 使用 docker 启动 mongodb

```bash
$ mkdir db
$ docker run -it --rm -p 27017:27017 -v ${PWD}/db:/data/db  --name mongodb -e MONGO_INITDB_ROOT_USERNAME=root -e MONGO_INITDB_ROOT_PASSWORD=123456 mongo:4.4
```

## 安装 mongodb-shell

```bash
 $ wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
 $ echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
 $ sudo apt-get update
 $ sudo apt-get install mongodb-org-shell
```

## 链接 mongodb

```bash
$ mongo --host 127.0.0.1 --port 27017 -u root -p 123456
MongoDB shell version v4.4.1
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("8a7be0c2-17a9-4a2e-9f43-dab0c52c02ca") }
MongoDB server version: 4.4.1
```

## 基本操作

```sql
> show databases;
admin   0.000GB
config  0.000GB
local   0.000GB

> use testdb
switched to db testdb

> db.createCollection("test");
{ "ok" : 1 }

> show collections;
test

> db.test.insert({
... _id : ObjectId("507f191e810c19729de860ea"),
... title: "MongoDB Overview",
... description: "MongoDB is no sql database",
... by: "tutorials point",
... url: "http://www.tutorialspoint.com",
... tags: ['mongodb', 'database', 'NoSQL'],
... likes: 100
... })
WriteResult({ "nInserted" : 1 })

> db.test.find()
{ "_id" : ObjectId("507f191e810c19729de860ea"), "title" : "MongoDB Overview", "description" : "MongoDB is no sql database", "by" : "tutorials point", "url" : "http://www.tutorialspoint.com", "tags" : [ "mongodb", "database", "NoSQL" ], "likes" : 100 }

> db.test.find().pretty()
{
	"_id" : ObjectId("507f191e810c19729de860ea"),
	"title" : "MongoDB Overview",
	"description" : "MongoDB is no sql database",
	"by" : "tutorials point",
	"url" : "http://www.tutorialspoint.com",
	"tags" : [
		"mongodb",
		"database",
		"NoSQL"
	],
	"likes" : 100
}
```

## 参考资料
- <https://hub.docker.com/_/mongo>
- <https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/#install-mongodb-community-edition-using-deb-packages>
