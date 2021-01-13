---
title:  "etcd 集群服务测试"
author: adream307
date:   2020-10-29 15:48:00 +0800
categories: [etcd]
tags: [etcd]
---

## docker-compose
```yml
version: '3.5'
services:
    machine-1:
        image: quay.io/coreos/etcd:v3.4.9
        hostname: machine-1
        container_name: machine-1
        command: etcd --data-dir=data.etcd --name machine-1 --initial-advertise-peer-urls http://machine-1:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://machine-1:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster machine-1=http://machine-1:2380,machine-2=http://machine-2:2380,machine-3=http://machine-3:2380 --initial-cluster-state new --initial-cluster-token etcd-token-01
        ports:
            - 10001:2379
        networks:
            - etcd-net

    machine-2:
        image: quay.io/coreos/etcd:v3.4.9
        hostname: machine-2
        container_name: machine-2
        command: etcd --data-dir=data.etcd --name machine-2 --initial-advertise-peer-urls http://machine-2:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://machine-2:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster machine-1=http://machine-1:2380,machine-2=http://machine-2:2380,machine-3=http://machine-3:2380 --initial-cluster-state new --initial-cluster-token etcd-token-01
        ports:
            - 10002:2379
        networks:
            - etcd-net

    machine-3:
        image: quay.io/coreos/etcd:v3.4.9
        hostname: machine-3
        container_name: machine-3
        command: etcd --data-dir=data.etcd --name machine-3 --initial-advertise-peer-urls http://machine-3:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://machine-3:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster machine-1=http://machine-1:2380,machine-2=http://machine-2:2380,machine-3=http://machine-3:2380 --initial-cluster-state new --initial-cluster-token etcd-token-01
        ports:
            - 10003:2379
        networks:
            - etcd-net

    client:
        image: quay.io/coreos/etcd:v3.4.9
        hostname: etcd-client
        container_name: etcd-client
        environment:
            - ETCDCTL_API=3
            - ENDPOINTS=machine-1:2379,machine-2:2379,machine-3:2379
        command: sh
        stdin_open: true # docker run -i
        tty: true        # docker run -t
        networks:
            - etcd-net
                  
networks:
    etcd-net:
        name: etcd-net
```

## 启动 etcd 集群
```bash
docker-compose up -d
```

## 观察集群状态
进入 `etcd-client`
```bash
docker exec -it etcd-client sh
```
观察集群状态
```bash
$ etcdctl --endpoints=$ENDPOINTS endpoint health
machine-1:2379 is healthy: successfully committed proposal: took = 2.071823ms
machine-3:2379 is healthy: successfully committed proposal: took = 2.167931ms
machine-2:2379 is healthy: successfully committed proposal: took = 2.16156ms
```

## 关闭 `machine-1` 节点
```bash
docker stop machine-1
docker rm machine-1
```

## 再次观察集群状态
```bash
$ etcdctl --endpoints=$ENDPOINTS endpoint health
{"level":"warn","ts":"2020-07-28T06:47:37.207Z","caller":"clientv3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"endpoint://client-68be9f9c-ccdb-4f36-9fb4-81be4d9af349/machine-1:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: all SubConns are in TransientFailure, latest connection error: connection error: desc = \"transport: Error while dialing dial tcp: lookup machine-1 on 127.0.0.11:53: no such host\""}
machine-3:2379 is healthy: successfully committed proposal: took = 1.734529ms
machine-2:2379 is healthy: successfully committed proposal: took = 1.688258ms
machine-1:2379 is unhealthy: failed to commit proposal: context deadline exceeded
Error: unhealthy cluster
```

## 从 etcd 集群中移除 `machine-1` 节点
获取节点 `ID`
```bash
$ etcdctl --endpoints=$ENDPOINTS member list
65b33e44da2c9667, started, machine-3, http://machine-3:2380, http://machine-3:2379, false
7260c0ff9b1fabc6, started, machine-1, http://machine-1:2380, http://machine-1:2379, false
d142e430c3316fa8, started, machine-2, http://machine-2:2380, http://machine-2:2379, false
``` 

移除节点
```bash
$ etcdctl --endpoints=$ENDPOINTS member remove 7260c0ff9b1fabc6
Member 7260c0ff9b1fabc6 removed from cluster fde02badc8c9071d
```

节点状态
```bash
$ etcdctl --endpoints=machine-2:2379 member list
65b33e44da2c9667, started, machine-3, http://machine-3:2380, http://machine-3:2379, false
d142e430c3316fa8, started, machine-2, http://machine-2:2380, http://machine-2:2379, false

$ etcdctl --endpoints=machine-2:2379,machine-3:2379 endpoint health
machine-3:2379 is healthy: successfully committed proposal: took = 2.063171ms
machine-2:2379 is healthy: successfully committed proposal: took = 1.972414ms
```

## 在 etcd 集群重新添加 `machine-1`
添加 `machine-1`
```bash
$ etcdctl --endpoints=machine-2:2379,machine-3:2379 member add machine-1 --peer-urls=http://machine-1:2380
Member 8f29007851c95572 added to cluster fde02badc8c9071d

ETCD_NAME="machine-1"
ETCD_INITIAL_CLUSTER="machine-3=http://machine-3:2380,machine-1=http://machine-1:2380,machine-2=http://machine-2:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://machine-1:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

查看 `machine-1`
```bash
$ etcdctl --endpoints=machine-2:2379 member list
65b33e44da2c9667, started, machine-3, http://machine-3:2380, http://machine-3:2379, false
8f29007851c95572, unstarted, , http://machine-1:2380, , false
d142e430c3316fa8, started, machine-2, http://machine-2:2380, http://machine-2:2379, false
```

启动 `machine-1`
```bash
docker run -d --name machine-1 --hostname machine-1 --net etcd-net -p 10001:2379 quay.io/coreos/etcd:v3.4.9  \
etcd --data-dir=data.etcd --name machine-1 \
	 --initial-advertise-peer-urls http://machine-1:2380 \
	 --listen-peer-urls http://0.0.0.0:2380 \
	 --advertise-client-urls http://machine-1:2379 \
	 --listen-client-urls http://0.0.0.0:2379 \
	 --initial-cluster machine-3=http://machine-3:2380,machine-1=http://machine-1:2380,machine-2=http://machine-2:2380 \
	 --initial-cluster-state existing \
	 --initial-cluster-token etcd-token-01
```

再次查看节点状态
```
$ etcdctl --endpoints=machine-2:2379 member list
65b33e44da2c9667, started, machine-3, http://machine-3:2380, http://machine-3:2379, false
8f29007851c95572, started, machine-1, http://machine-1:2380, http://machine-1:2379, false
d142e430c3316fa8, started, machine-2, http://machine-2:2380, http://machine-2:2379, false

$ etcdctl --endpoints=$ENDPOINTS endpoint health
machine-2:2379 is healthy: successfully committed proposal: took = 1.522126ms
machine-3:2379 is healthy: successfully committed proposal: took = 1.908916ms
machine-1:2379 is healthy: successfully committed proposal: took = 1.859882ms
```

## 插入数据
```bash
$ etcdctl --endpoints=$ENDPOINTS put key1 value1
OK
$ etcdctl --endpoints=$ENDPOINTS get key1
key1
value1
```

## 列出所有的 `key`
```bash
$ etcdctl --endpoints=$ENDPOINTS get '' --prefix
key1
value1
key2
value2
key3
value3
```

## 监视 `etcd` 所有 `key`
```bash
$ etcdctl --endpoints=$ENDPOINTS watch '' --prefix
```
另外开一个 `shell` 界面，插入及删除数据
```bash
$ etcdctl --endpoints=$ENDPOINTS put key4 value4
OK
$ etcdctl --endpoints=$ENDPOINTS del key4
1
```
`watch` 界面输出如下
```bash
$ etcdctl --endpoints=$ENDPOINTS watch '' --prefix
PUT
key4
value4
DELETE
key4
```

## 关闭 etcd 集群
```bash
docker stop machine-1
docker rm machine-1
docker-compose down
```


