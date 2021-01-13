---
title:  "etcd 分布式锁实现原理"
author: adream307
date:   2020-11-02 14:55:00 +0800
categories: [etcd]
tags: [etcd]
---

## docker 启动 etcd
```bash
docker run --rm -p 2379:2379 -p 2380:2380 --name etcd \
quay.io/coreos/etcd:v3.4.9 etcd \
--name node1 \
--initial-advertise-peer-urls http://127.0.0.1:2380 \
--listen-peer-urls http://0.0.0.0:2380 \
--advertise-client-urls http://127.0.0.1:2379 \
--listen-client-urls http://0.0.0.0:2379 \
--initial-cluster node1=http://127.0.0.1:2380
```

## docker-compose 启动 etcd
```yml
version: '3'
services:
    etcd-1:
        image: quay.io/coreos/etcd:v3.4.9
        hostname: etcd-1
        container_name: etcd-1
        command: etcd --data-dir=data.etcd --name etcd-1 --initial-advertise-peer-urls http://etcd-1:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://etcd-1:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster etcd-1=http://etcd-1:2380 --initial-cluster-state new --initial-cluster-token etcd-token-01
        ports:
            - 2379:2379
            - 2380:2380
        networks:
            - etcd-net
networks:
    etcd-net:
```

## my_mutex
```go
package main

import (
	"context"
	"fmt"
	"go.etcd.io/etcd/clientv3"
	"go.etcd.io/etcd/mvcc/mvccpb"
	"log"
	"path"
	"strconv"
	"sync"
)

type myMutex struct {
	client  *clientv3.Client
	leaseId clientv3.LeaseID
	pfx     string
	myKey   string
	myRev   int64
}

func newMyMutex(cli *clientv3.Client, pfx string) (*myMutex, error) {
	resp, err := cli.Grant(context.TODO(), 15)
	if err != nil {
		return nil, err
	}
	if _, err := cli.KeepAlive(context.TODO(), resp.ID); err != nil {
		return nil, err
	}

	mu := &myMutex{
		client:  cli,
		leaseId: resp.ID,
		pfx:     pfx,
		myKey:   path.Join(pfx, strconv.FormatInt(int64(resp.ID), 16)),
		myRev:   -1,
	}

	return mu, nil
}

func (mu *myMutex) Close() error {
	ctx, cancel := context.WithCancel(context.TODO())
	_, err := mu.client.Revoke(ctx, mu.leaseId)
	cancel()
	return err
}

func (mu *myMutex) UnLock(ctx context.Context) error {
	_, err := mu.client.Delete(ctx, mu.myKey)
	return err
}

func (mu *myMutex) Lock(ctx context.Context) error {
	cmp := clientv3.Compare(clientv3.CreateRevision(mu.myKey), "=", 0)
	put := clientv3.OpPut(mu.myKey, "", clientv3.WithLease(mu.leaseId))
	get := clientv3.OpGet(mu.myKey)
	getOwner := clientv3.OpGet(mu.pfx, clientv3.WithFirstCreate()...)
	resp, err := mu.client.Txn(ctx).If(cmp).Then(put, getOwner).Else(get, getOwner).Commit()
	if err != nil {
		return err
	}
	mu.myRev = resp.Header.Revision
	if !resp.Succeeded {
		mu.myRev = resp.Responses[0].GetResponseRange().Kvs[0].CreateRevision
	}
	ownerRev := resp.Responses[1].GetResponseRange().Kvs[0].CreateRevision
	if ownerRev == mu.myRev {
		return nil
	}
	getOps := append(clientv3.WithLastCreate(), clientv3.WithMaxCreateRev(mu.myRev-1))
	for {
		resp, err := mu.client.Get(ctx, mu.pfx, getOps...)
		if err != nil {
			return err
		}
		if len(resp.Kvs) == 0 {
			break
		}
		wch := mu.client.Watch(ctx, string(resp.Kvs[0].Key), clientv3.WithRev(resp.Header.Revision))
		for wr := range wch {
			isBreak := false
			for _, ev := range wr.Events {
				if ev.Type == mvccpb.DELETE {
					isBreak = true
					break
				}
			}
			if isBreak {
				break
			}
		}
	}
	return nil
}

func main() {
	var cnt int64
	var wg sync.WaitGroup
	endpoints := []string{"127.0.0.1:2379"}
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
			if err != nil {
				log.Fatal(err)
			}
			defer cli.Close()
			mu, err := newMyMutex(cli, "/my_mutex")
			if err != nil {
				log.Fatal(err)
			}
			defer mu.Close()
			mu.Lock(context.TODO())

			defer mu.UnLock(context.TODO())

			for k := 0; k < 100000; k++ {
				cnt = cnt + 1
			}
			wg.Done()
		}()
	}
	wg.Wait()
	fmt.Println(cnt)
}

```

## my_mutex 工作原理
1. 每个 `mutex` 会创建一个租约 `lease`，并且租约是长期有效的
2. 使用 `prefix/leaseId` 作为 `key` 向 `etcd` 插入数据
3. `etcd` 确保每次插入的 `key` 都有一个位移的 `CreateRevision`，并且 `CreateRevision` 是从 `0` 开始递增的
4. 第 `2` 条规定所有 `mutex` 按照 `prefix/leaseId` 作为 `key` 向 `etcd` 插入数据
5. 每个 `mutex` 监视 `prefix` 下所有 `key`,并且获得这些 `key` 的 `CreateRevision`
6. 如果 `mutex` 发现自己的 `prefix/leaseId` 的 `CreateRevision` 是这些 `prefix` 下所有 `key` 中最小的，那么当前 `mutex` 获得锁
