---
title:  "使用 Txn 一次性插入多个语句"
author: adream307
date:   2020-10-29 10:00:00 +0800
categories: [etcd]
tags: [etcd]
---

`etcd` 使用 `Txn` 提供简单的事务处理，使用这个特性，可以一次性插入多条语句，测试代码如下:
```go
package main

import (
	"context"
	"fmt"
	"go.etcd.io/etcd/clientv3"
	"log"
)

func main() {
	endpoints := []string{"127.0.0.1:10001", "127.0.0.1:10002", "127.0.0.1:10003"}
	cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	_, err = cli.Txn(context.TODO()).If().Then(
		clientv3.OpPut("k30", "v30"),
		clientv3.OpPut("k40", "v40"),
	).Commit()
	if err != nil {
		log.Fatal(err)
	}
	resp, err := cli.Txn(context.TODO()).If().Then(
		clientv3.OpGet("k30"),
		clientv3.OpGet("k40"),
	).Commit()
	if err != nil {
		log.Fatal(err)
	}
	for _, rp := range resp.Responses {
		for _, ev := range rp.GetResponseRange().Kvs {
			fmt.Printf("%s -> %s, create revision = %d\n",
				string(ev.Key),
				string(ev.Value),
				ev.CreateRevision)
		}
	}
}
```
程序输出代码如下：
```txt
k30 -> v30, create revision = 13
k40 -> v40, create revision = 13
```
`k30` 和`k40` 的 `create revision` 都是 `13`，表明这两个语句是一次性插入的