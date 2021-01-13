---
title:  "etcd revision 测试"
author: adream307
date:   2020-10-29 16:20:00 +0800
categories: [etcd]
tags: [etcd]
---

## docker-compose
启动 `etcd` 集群
```bash
docker-compose -f etcd-only.yml up -d
```
`etcd-only.yml`,设置 snapshot 为100，即每 `100` 个修改，产生一次 `snapshot`，清除 `WAL`
```yml
version: '3.5'
services:
    machine-1:
        image: quay.io/coreos/etcd:v3.4.9
        hostname: machine-1
        container_name: machine-1
        command: etcd --data-dir=data.etcd --name machine-1 --initial-advertise-peer-urls http://machine-1:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://machine-1:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster machine-1=http://machine-1:2380,machine-2=http://machine-2:2380,machine-3=http://machine-3:2380 --initial-cluster-state new --initial-cluster-token etcd-token-01 --snapshot-count 100
        ports:
            - 10001:2379
        networks:
            - etcd-net

    machine-2:
        image: quay.io/coreos/etcd:v3.4.9
        hostname: machine-2
        container_name: machine-2
        command: etcd --data-dir=data.etcd --name machine-2 --initial-advertise-peer-urls http://machine-2:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://machine-2:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster machine-1=http://machine-1:2380,machine-2=http://machine-2:2380,machine-3=http://machine-3:2380 --initial-cluster-state new --initial-cluster-token etcd-token-01 --snapshot-count 100
        ports:
            - 10002:2379
        networks:
            - etcd-net

    machine-3:
        image: quay.io/coreos/etcd:v3.4.9
        hostname: machine-3
        container_name: machine-3
        command: etcd --data-dir=data.etcd --name machine-3 --initial-advertise-peer-urls http://machine-3:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://machine-3:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster machine-1=http://machine-1:2380,machine-2=http://machine-2:2380,machine-3=http://machine-3:2380 --initial-cluster-state new --initial-cluster-token etcd-token-01 --snapshot-count 100
        ports:
            - 10003:2379
        networks:
            - etcd-net
                  
networks:
    etcd-net:
        name: etcd-net
```

## 获得节点 `IP` 地址
```bash
$ IP1=$(docker exec machine-1 cat /etc/hosts | grep "machine-1" | awk '{print $1}')
$ IP2=$(docker exec machine-2 cat /etc/hosts | grep "machine-2" | awk '{print $1}')
$ IP3=$(docker exec machine-3 cat /etc/hosts | grep "machine-3" | awk '{print $1}')
$ ENDPOINTS=$IP1:2379,$IP2:2379,$IP3:2379
$ echo "endpoints : $ENDPOINTS"
endpoints : 192.168.208.2:2379,192.168.208.3:2379,192.168.208.4:2379
```

## 查看当前 `revision`
向`etcd`查询某个 `key` 即可获得当前 `revision`，因此最简单的办法就是查询一个不存在的 `key`。

如下所示，查询 `revisiontestkey` ，因为这个 `key` 不存在，那么只返回查询的 `head`, `revision` 就在 `head` 中
```
$ etcdctl --endpoints=$ENDPOINTS get revisiontestkey -w json
{"header":{"cluster_id":18293669711776909085,"member_id":15078865400474202024,"revision":1,"raft_term":2}}
```

## `snapshot`
进入 `machine-1` 观察 `snapshot` 文件夹
```bash
docker exec -it machine-1 sh
```
观察 `snapshot` 文件夹
```bash
$ ls data.etcd/member/snap -lh
total 20K
-rw------- 1 root root 20K Aug  4 07:56 db
```
当前 `snapshot` 文件夹内只有 `db` 文件，并未生成 `snapshot`

## 插入数据
因为 `snapshot-count` 这是为 `100`，那么插入 `150` 个数据，中间一定定触发 `snapshot`
```bash
CNT=0
while [ $CNT -lt 150 ]; do
    etcdctl --endpoints=$ENDPOINTS put key${CNT} value${CNT}
    CNT=$[$CNT + 1]
done
```
插入数据后，再次查询 `revision`，变成了 `151`
```bash
$ etcdctl --endpoints=$ENDPOINTS get revisiontestkey -w json
{"header":{"cluster_id":18293669711776909085,"member_id":8241799522139745222,"revision":151,"raft_term":2}}
```

## 再次查看 `snapshot`
可以观察到生成的 `snapshot` 文件
```bash
$ ls data.etcd/member/snap -lh
total 56K
-rw-r--r-- 1 root root 8.9K Aug  4 08:26 0000000000000002-0000000000000065.snap
-rw------- 1 root root  40K Aug  4 08:26 db
```

## 根据 `revision` 查询
查询 `revision` 为 `5` 之前的所有数据
```bash
$ etcdctl --endpoints=$ENDPOINTS get '' --prefix --rev 5    
key0
value0
key1
value1
key2
value2
key3
value3
```
因为`snapshot-count` 为 `100`，那么在 插入 `key99` 后，`etcd` 生成 `snapshot`，而 `key99` 对应的 `revision` 为 `101`；但是在 `revision` 为 `101` 位置触发了 `snapshot` 操作后依然可以查询 `revision` 为 `5` 的数据，这说明`snapshot` 操作仅仅删除 `WAL`，并不会在 `revision` 上做任何删除操作

## 压缩 `revision`
将 `revision` 压缩，意味着删除指定 `revision` 之前的所有数据版本信息，如下所示，删除 `revision` 5 之前的所有版本信息
```bash
etcdctl --endpoints=$ENDPOINTS compaction 5
```
压缩后，可以列出 `revision` 为 `5` 之前的所有数据
```bash
$ etcdctl --endpoints=$ENDPOINTS get '' --prefix --rev 5
key0
value0
key1
value1
key2
value2
key3
value3
```
但是没办法列出 `revision` 为 `4` 之前的所欲数据
```bash
$ etcdctl --endpoints=$ENDPOINTS get '' --prefix --rev 4
{"level":"warn","ts":"2020-08-04T16:46:30.923+0800","caller":"clientv3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"endpoint://client-46e1b5a7-2fa4-4079-98df-ebddbf6896fa/192.168.208.2:2379","attempt":0,"error":"rpc error: code = OutOfRange desc = etcdserver: mvcc: required revision has been compacted"}
Error: etcdserver: mvcc: required revision has been compacted
```
提示我们需要的版本信息已经被删除了

## `revision` 测试
```bash
$ etcdctl --endpoints=$ENDPOINTS get key10 -w json 
{"header":{"cluster_id":18293669711776909085,"member_id":15078865400474202024,"revision":151,"raft_term":2},"kvs":[{"key":"a2V5MTA=","create_revision":12,"mod_revision":12,"version":1,"value":"dmFsdWUxMA=="}],"count":1}
```
查询的 `key10`，返回只的 `key` 和 `value` 使用 `base64` 编码，可以在这个网址:https://www.base64decode.org/ 进行解码

- `create_revision` 表示 `key10` 创建时的 `revision` 值;
- `mod_revision` 表示 `key10` 最后一次修改时的 `revision` 值，第一次创建时，这个值就是 `create_revision` 的值
- `version` 表示当前版本号,第一次创建时，改版本号为 `1`
```json
{
    "key":"a2V5MTA=",
    "create_revision":12,
    "mod_revision":12,
    "version":1,
    "value":"dmFsdWUxMA=="
}
```

修改 `key10`，并再次查询 `key10`
```bash
$ etcdctl --endpoints=$ENDPOINTS put key10 new_value10
OK
$ etcdctl --endpoints=$ENDPOINTS get key10 -w json
{"header":{"cluster_id":18293669711776909085,"member_id":8241799522139745222,"revision":152,"raft_term":2},"kvs":[{"key":"a2V5MTA=","create_revision":12,"mod_revision":152,"version":2,"value":"bmV3X3ZhbHVlMTA="}],"count":1}
```
- `create_revision` 的值不变，依然为 `12`
- `mod_revision` 变为当前最新的 `revision` 152
- `version` 变为 `2`
```json
{
    "key":"a2V5MTA=",
    "create_revision":12,
    "mod_revision":152,
    "version":2,
    "value":"bmV3X3ZhbHVlMTA="
}
```

删除所有 `key` 打头的数据
```bash
$ etcdctl --endpoints=$ENDPOINTS del key --prefix -w json
{"header":{"cluster_id":18293669711776909085,"member_id":15078865400474202024,"revision":153,"raft_term":2},"deleted":150}
```

查询所有 `key` 打头的数据，因为所有的数据均被删除，所以返回的数据为空
```bash
etcdctl --endpoints=$ENDPOINTS get key --prefix -w json
{"header":{"cluster_id":18293669711776909085,"member_id":8241799522139745222,"revision":153,"raft_term":2}}
```

多次修改值
```bash
$ etcdctl --endpoints=$ENDPOINTS put key150 value150
$ etcdctl --endpoints=$ENDPOINTS put key150 new_150 -w json
{"header":{"cluster_id":18293669711776909085,"member_id":15078865400474202024,"revision":155,"raft_term":2}}
$ etcdctl --endpoints=$ENDPOINTS put key150 new_new_150 -w json
{"header":{"cluster_id":18293669711776909085,"member_id":8241799522139745222,"revision":156,"raft_term":2}}
$ etcdctl --endpoints=$ENDPOINTS put key150 new_new_new_150 -w json
{"header":{"cluster_id":18293669711776909085,"member_id":7328269484100982375,"revision":157,"raft_term":2}}
```

按照 `revision` 获得值
```bash
$ etcdctl --endpoints=$ENDPOINTS get key150 --rev 156
key150
new_new_150
$ etcdctl --endpoints=$ENDPOINTS get key150 --rev 155
key150
new_150
$ etcdctl --endpoints=$ENDPOINTS get key150 --rev 154
key150
value150
```
如果某个值有多个版本，那么根据 `revision` 获得的值为离指定 `revision` 最近的的版本


## 关闭 `etcd` 集群
```bash
docker-compose -f etcd-only.yml down
```

