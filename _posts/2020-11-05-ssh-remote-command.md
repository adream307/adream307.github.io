---
title:  "ssh 执行远程命令和端口转发"
author: adream307
date:   2020-11-05 14:17:00 +0800
categories: [linux,ssh]
tags: [ssh]
---

## 执行远程命令

查看远程主机是否运行进程httpd
```bash
ssh user@remote_host 'ps ax | grep httpd'
```

## 绑定本地端口

假定我们要让 `1081` 端口的数据，都通过 `SSH` 传向远程主机，命令就这样写
```bash
ssh -D 1081 user@remote_host
```
`SSH` 会建立一个 `socket`，去监听本地的 `1081` 端口，所有链接本地 `1081` 端口的数据都会被转发到 `remote_host`

如果 `remote_host` 具备翻墙的功能，那么这个命令相当于在本地的 `1081` 端口上建立一个 `sock` 的代理服务器

使用 `netstat` 命令观察在 `localhost:1081` 上建立 `tcp` 监听端口 
```bash
sudo netstat -lt     
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:ssh             0.0.0.0:*               LISTEN      
tcp        0      0 localhost:1081          0.0.0.0:*               LISTEN   
```

在本机的所有网卡上监听 `1081` 端口
```bash
ssh -D 0.0.0.0:1081 user@remote_host
```

```bash
ssh -o GatewayPorts=true -D 1081 user@remote_host
```

使用 `netstat` 命令观察

```bash
sudo netstat -lt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:ssh             0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:1081            0.0.0.0:*               LISTEN
```


## 本地端口转发

有时，绑定本地端口还不够，还必须指定数据传送的目标主机，从而形成点对点的 `端口转发` 

为了区别后文的 `远程端口转发` ，我们把这种情况称为 `本地端口转发（Local forwarding）`

假定 `host1` 是本地主机，`host2` 是远程主机。由于种种原因，这两台主机之间无法连通

但是，另外还有一台 `host3`，可以同时连通前面两台主机。因此，很自然的想法就是，通过 `host3`，将 `host1` 连上 `host2`

我们在 `host1` 执行下面的命令：
```bash
ssh -L 1081:host2:1080 host3
```

命令中的 `L` 参数一共接受三个值，分别是 `本地端口:目标主机:目标主机端口`，它们之间用冒号分隔

这条命令的意思，就是指定 `SSH` 绑定本地端口 `1081`，然后指定 `host3` 将所有的数据，转发到目标主机 `host2` 的 `1080` 端口

这样一来，我们只要连接 `host1` 的 `1081` 端口，就等于连上了 `host2` 的 `1080` 端口

`本地端口转发` 使得 `host1` 和 `host3` 之间仿佛形成一个数据传输的秘密隧道，因此又被称为 `SSH隧道`

在 `host1`上 使用 `netstat` 命令观察

```bash
sudo netstat -lt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State         
tcp        0      0 0.0.0.0:ssh             0.0.0.0:*               LISTEN        
tcp        0      0 localhost:1081          0.0.0.0:*               LISTEN
```

在 `host1` 的所有网卡上监听 `1081` 端口
```bash
ssh -L 0.0.0.0:1081:host2:1080 host3
```
或
```bash
ssh -o GatewayPorts=true -L 1081:host2:1080 host3
```

使用 `netstat` 命令观察

```bash
sudo netstat -lt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:ssh             0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:1081            0.0.0.0:*               LISTEN
```

## 远程端口转发

既然 `本地端口转发` 是指绑定本地端口的转发，那么 `远程端口转发（remote forwarding）` 当然是指绑定远程端口的转发

还是接着看上面那个例子，`host1` 与 `host2` 之间无法连通，必须借助 `host3` 转发

但是，特殊情况出现了，`host3` 是一台内网机器，它可以连接外网的 `host1`，但是反过来就不行，外网的 `host1` 连不上内网的 `host3`

这时，`本地端口转发` 就不能用了，怎么办？

解决办法是，既然 `host3` 可以连 `host1`，那么就从 `host3` 上建立与 `host1` 的 `SSH` 连接，然后在 `host1`上使用这条连接就可以了。

我们在 `host3` 执行下面的命令：

```bash
ssh -R 1081:host2:1080 host1
```

`R` 参数也是接受三个值，分别是 `远程主机端口:目标主机:目标主机端口` 

这条命令的意思，就是让 `host1` 监听它自己的 `1081` 端口，然后将所有数据经由 `host3`，转发到 `host2` 的 `1080`端口

由于对于 `host3` 来说，`host1` 是远程主机，所以这种情况就被称为 `远程端口绑定`

绑定之后，我们在 `host1` 就可以连接 `host2` 了

这里必须指出，`远程端口转发` 的前提条件是，`host1` 和 `host3` 两台主机都有 `sshD` 和 `ssh` 客户端

在 `host1`上 使用 `netstat` 命令观察

```bash
sudo netstat -lt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State         
tcp        0      0 0.0.0.0:ssh             0.0.0.0:*               LISTEN        
tcp        0      0 localhost:1081          0.0.0.0:*               LISTEN
```

在 `host1` 的所有网卡上监听 `1081` 端口

在 `host1` 的 `/etc/ssh/sshd_config` 文件下添加, `user-name` 为 ssh 登录 `host1` 的用户名
```bash
Match User <user-name>
        GatewayPorts yes
```

我们在 `host3` 执行下面的命令：

```bash
ssh -R 1081:host2:1080 host1
```

在 `host1`上 使用 `netstat` 命令观察

```bash
sudo netstat -lt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address       State
tcp        0      0 0.0.0.0:ssh             0.0.0.0:*             LISTEN
tcp        0      0 0.0.0.0:1081            0.0.0.0:*             LISTEN
```

## 参考链接
- <https://stackoverflow.com/questions/23781488/how-to-make-ssh-remote-port-forward-that-listens-0-0-0-0>
- <http://www.ruanyifeng.com/blog/2011/12/ssh_port_forwarding.html>


