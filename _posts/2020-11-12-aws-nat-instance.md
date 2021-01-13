---
title:  "AWS NAT 实例"
author: adream307
date:   2020-11-12 21:31:00 +0800
categories: [aws,nat]
tags: [aws,nat]
---
## 创建 `VPC`
`vpc` 地址范围为 `192.168.0.0/16`

![create vpc](/assets/img/aws-nat-instance/create-vpc.png)

新建 `vpc` 详细信息如下
![vpc detail](/assets/img/aws-nat-instance/vpc-detail.png)

## 创建两个子网
`M0-0` : `192.168.0.0/24`

![subnet m0-0](/assets/img/aws-nat-instance/subnet-m0-0.png)

`M0-1` : `192.168.1.0/24`

![subnet m0-0](/assets/img/aws-nat-instance/subnet-m0-1.png)

子网详细信息

![subnet detail](/assets/img/aws-nat-instance/subnet-detail.png)

## 创建互联网网关
![internet gateway](/assets/img/aws-nat-instance/internet-gateway.png)

选择  `Actions` -> `Attach to VPC` 将网关附加到 `VPC M0`

![internet gateway detail](/assets/img/aws-nat-instance/internet-gateway-detail.png)

网关附加到 `VPC M0` 

![attatch igw to vpc](/assets/img/aws-nat-instance/attatch-igw-to-vpc.png)

## 新建立路由表
![route table](/assets/img/aws-nat-instance/route-table.png)

新建立的路由表默认只有一条规则，用于 `VPC` 内部

![route default rule](/assets/img/aws-nat-instance/route-default-rule.png)

添加一条路由规则，将流向 `VPC` 外的流量发送到互联网网关

- `Destination` : `0.0.0.0/0`
- `Target` : 之前创建的互联网网关

![route outside rule](/assets/img/aws-nat-instance/route-outside-rule.png)

添加规则后的路由表

![route rule detail](/assets/img/aws-nat-instance/route-rule-detail.png)

将路由表关联 `M0-0` 子网，使得该子网能够访问公网

![associate subnet](/assets/img/aws-nat-instance/associate-subnet.png)

关联子网后的路由表

![subnet association](/assets/img/aws-nat-instance/subnet-association.png)

## 创建公网 `Security Group`

在 `VPC M0` 上创建一个 `Security Group` 

![sg-m0.png](/assets/img/aws-nat-instance/sg-m0.png)


`Security Group` 的规则如下
- 允许来自当前开发机器的所有 `TCP` 链接
- 允许来自 `VPC` 内部的 `ICMP-IPv4`，用于 `ping` 测试

![sg-rules.png](/assets/img/aws-nat-instance/sg-rules.png)


`Security Group` 详细信息

![sg-m0-detail.png](/assets/img/aws-nat-instance/sg-m0-detail.png)


## 创建 `NAT` 实例

`AMI` 选择最新的 `NAT Image` 

![nat-ami.png](/assets/img/aws-nat-instance/nat-ami.png)


选择实例类型，本次演示选择 `t2.micro`

![nat-inst-type.png](/assets/img/aws-nat-instance/nat-inst-type.png)


配置实例：
- `Network` : 选择 `VPC M0`
- `Subnet` : 选择 `M0-0`
- `Auto-assign Public IP` : `Enable`
  
![nat-inst-conf.png](/assets/img/aws-nat-instance/nat-inst-conf.png)


添加标签 `Name` -> `M0-0-0`

![nat-m0-0-0.png](/assets/img/aws-nat-instance/nat-m0-0-0.png)


`Security Group` 选择之前创建的 `M0`

![nat-sg-m0.png](/assets/img/aws-nat-instance/nat-sg-m0.png)


选择一个 `key pair`，然后启动实例

![nat-key-pair.png](/assets/img/aws-nat-instance/nat-key-pair.png)


## 禁用 `NAT` 实例的 `SrcDestCheck` 属性

依次选择 `NAT 实例 M0-0-0` -> `Actions` -> `Networking` -> `Change Source/Dest.Check`

![src-dest-check.png](/assets/img/aws-nat-instance/src-dest-check.png)

选择 `Yes, Disable`

![src-dest-check-disable.png](/assets/img/aws-nat-instance/src-dest-check-disable.png)


`NAT` 实例详细信息
- 公网 `IP` 为 `68.79.5.240`

![m0-0-0-detail.png](/assets/img/aws-nat-instance/m0-0-0-detail.png)

## `SSH` 远程登录 `M0-0-0`
```bash
$ ssh -i ~/.ssh/aws_passed.pem ec2-user@68.79.5.240
The authenticity of host '68.79.5.240 (68.79.5.240)' can't be established.
ECDSA key fingerprint is SHA256:Z14wTfP8Yl2nCuAASW4LXyNflgTOEtnR46dtsKGZ2Zo.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '68.79.5.240' (ECDSA) to the list of known hosts.

       __|  __|_  )
       _|  (     /   Amazon Linux AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-ami/2018.03-release-notes/
[ec2-user@ip-192-168-0-6 ~]$ ping www.baidu.com
PING www.a.shifen.com (220.181.38.150) 56(84) bytes of data.
64 bytes from 220.181.38.150: icmp_seq=1 ttl=46 time=20.2 ms
64 bytes from 220.181.38.150: icmp_seq=2 ttl=46 time=20.2 ms
64 bytes from 220.181.38.150: icmp_seq=3 ttl=46 time=20.3 ms

```

## 更新 `VPC M0` 主路由表

选择 `M0` 的主路由表，添加规则：流向外网的所有流量路由到 `NAT` 实例
- `Destination` : `0.0.0.0/0` 流向外网的所有流量
- `Target` : 选择创建的 `NAT` 实例 `M0-0-0`

![m0-main-table.png](/assets/img/aws-nat-instance/m0-main-table.png)


主路由表详细信息

![m0-main-table-detail.png](/assets/img/aws-nat-instance/m0-main-table-detail.png)


将主路由表关联到 `M0-1` 子网

![m0-main-table-subnet.png](/assets/img/aws-nat-instance/m0-main-table-subnet.png)


## 创建私网 `Security Group`

创建一个私网的 `Security Group`，允许所有来自 `M0-0` 子网的所有数据

![sg-m0-private.png](/assets/img/aws-nat-instance/sg-m0-private.png)

## 创建私网实例

创建私网实例，用于 `NAT` 测试
- `Network` : 选择 `VPC M0`
- `Subnet` : 选择 `M0-1`
- `Auto-assign Public IP` : `Disable`

![private-ec2-conf.png](/assets/img/aws-nat-instance/private-ec2-conf.png)

`Security Group` 选择创建的私网安全组 `M0-Private`

![private-ec2-sg.png](/assets/img/aws-nat-instance/private-ec2-sg.png)

私网实例详细信息如下
- 私网 IP 地址为 : `192.168.1.64`
- 没有公网 IP

![m0-1-0.png](/assets/img/aws-nat-instance/m0-1-0.png)

从 `NAT` 实例 `M0-0-0` ping `M0-1-0` 
```bash
[ec2-user@ip-192-168-0-6 ~]$ ping 192.168.1.64
PING 192.168.1.64 (192.168.1.64) 56(84) bytes of data.
64 bytes from 192.168.1.64: icmp_seq=1 ttl=64 time=0.495 ms
64 bytes from 192.168.1.64: icmp_seq=2 ttl=64 time=0.451 ms
64 bytes from 192.168.1.64: icmp_seq=3 ttl=64 time=0.442 ms
64 bytes from 192.168.1.64: icmp_seq=4 ttl=64 time=0.476 ms
64 bytes from 192.168.1.64: icmp_seq=5 ttl=64 time=0.486 ms

```

## 测试私网实例 `M0-1-0`

从 `NAT` 实例 `M0-0-0` ssh 进入 `M0-1-0` 
```bash
[ec2-user@ip-192-168-0-6 ~]$ ssh -i ~/.ssh/aws_passed.pem ubuntu@192.168.1.64
The authenticity of host '192.168.1.64 (192.168.1.64)' can't be established.
ECDSA key fingerprint is SHA256:PWhn4SijJrgQzU56d1+5sP/o3BR2/GwRq0JJieClNGU.
ECDSA key fingerprint is MD5:d5:d7:52:8d:0b:11:18:29:e4:1a:55:15:bc:8a:d1:d6.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.1.64' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-1009-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Nov 13 02:23:18 UTC 2020

  System load:  0.0               Processes:             99
  Usage of /:   16.2% of 7.69GB   Users logged in:       0
  Memory usage: 18%               IPv4 address for eth0: 192.168.1.64
  Swap usage:   0%

0 updates can be installed immediately.
0 of these updates are security updates.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
ubuntu@ip-192-168-1-64:~$
```

测试 `M0-1-0` 访问外网
```
ubuntu@ip-192-168-1-64:~$ ping www.baidu.com
PING www.a.shifen.com (220.181.38.149) 56(84) bytes of data.
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=1 ttl=45 time=21.9 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=2 ttl=45 time=22.0 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=3 ttl=45 time=21.9 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=4 ttl=45 time=22.0 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=5 ttl=45 time=21.9 ms

```

## 设置 `NAT` 端口转发

在 `NAT` 实例 `M0-0-0` 设置端口转发: 将 `M0-0-0` 的 `10022` 端口上的 `TCP` 数据转发到 `M0-1-0` 的 `22`端口 
```bash
[ec2-user@ip-192-168-0-6 ~]$ sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 10022 -j DNAT --to 192.168.1.64:22
[ec2-user@ip-192-168-0-6 ~]$ sudo iptables -t nat -L
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
DNAT       tcp  --  anywhere             anywhere             tcp dpt:10022 to:192.168.1.64:22

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  anywhere             anywhere            
[ec2-user@ip-192-168-0-6 ~]$ 
```

在本地开发机器上，通过 `NAT`实例 `M0-0-0` 的 `10022` 端口 `SSH` 登录 `M0-1-0`

```bash
$ ssh -i~/.ssh/aws_passed.pem -p 10022 ubuntu@68.79.5.240
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-1009-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Nov 13 02:33:55 UTC 2020

  System load:  0.0               Processes:             99
  Usage of /:   16.3% of 7.69GB   Users logged in:       0
  Memory usage: 18%               IPv4 address for eth0: 192.168.1.64
  Swap usage:   0%


0 updates can be installed immediately.
0 of these updates are security updates.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Fri Nov 13 02:23:19 2020 from 192.168.0.6
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@ip-192-168-1-64:~$ ping www.baidu.com
PING www.a.shifen.com (220.181.38.150) 56(84) bytes of data.
64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=1 ttl=45 time=20.6 ms
64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=2 ttl=45 time=20.7 ms
64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=3 ttl=45 time=20.8 ms
64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=4 ttl=45 time=20.7 ms

```


## 参考资料
- <https://docs.aws.amazon.com/zh_cn/vpc/latest/userguide/VPC_NAT_Instance.html>
- <https://www.cnblogs.com/Bourbon-tian/p/9004242.html>