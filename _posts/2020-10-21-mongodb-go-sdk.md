---
title:  "Mongodb go sdk 测试"
author: adream307
date:   2020-10-21 11:50:00 +0800
categories: [mongodb, go-sdk]
tags: [mongodb, go]
---

启动 `mongodb`
```bash
docker run -it --rm -p 27017:27017  --name mongodb -e MONGO_INITDB_ROOT_USERNAME=root -e MONGO_INITDB_ROOT_PASSWORD=123456 mongo:4.4
```

`mongo_exp1.go`
```go
package main

import (
	"context"
	"fmt"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
	"log"
	"time"
)

type testData struct {
	Name  string `json:"name" bson:"name"`
	Value string `json:"value" bson:"name"`
}

func main() {
	client, err := mongo.NewClient(options.Client().ApplyURI("mongodb://127.0.0.1:27017"),
		&options.ClientOptions{Auth: &options.Credential{
			AuthMechanism:           "",
			AuthMechanismProperties: nil,
			AuthSource:              "",
			Username:                "root",
			Password:                "123456",
			PasswordSet:             true,
		}})
	if err != nil {
		log.Fatal(err)
	}
	ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
	err = client.Connect(ctx)
	if err != nil {
		log.Fatal(err)
	}
	defer client.Disconnect(ctx)

	testDatabase := client.Database("testdb")
	testCollection := testDatabase.Collection("tests")

	td := testData{
		Name:  "InsertTestName1",
		Value: "InsertTetValue1",
	}

	if podcastResult, err := testCollection.InsertOne(ctx, td); err != nil {
		log.Fatal(err)
	} else {
		fmt.Println(podcastResult)
	}

	if cursor, err := testCollection.Find(ctx, bson.D{}); err != nil {
		log.Fatal(err)
	} else {
		for cursor.Next(ctx) {
			var od testData
			if err = cursor.Decode(&od); err != nil {
				log.Fatal(err)
			} else {
				fmt.Printf("%s : %s\n", od.Name, od.Value)
			}
		}
	}
}

```

程序输出
```txt
&{ObjectID("5f8fb0b3022b456b712e61e0")}
InsertTestName1 : InsertTetValue1
```

`shell` 登录 mongodb
```bash
$ mongo --host 127.0.0.1 --port 27017 -u root -p 123456

> use testdb;
switched to db testdb

> db.tests.find();
{ "_id" : ObjectId("5f8fb0b3022b456b712e61e0"), "name" : "InsertTestName1", "value" : "InsertTetValue1" }

> db.tests.find().pretty();
{
	"_id" : ObjectId("5f8fb0b3022b456b712e61e0"),
	"name" : "InsertTestName1",
	"value" : "InsertTetValue1"
}
```