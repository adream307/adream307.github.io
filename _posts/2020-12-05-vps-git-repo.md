---
title:  "在 vps 上 fork github 的 repo"
author: adream307
date:   2020-12-05 19:38:00 +0800
categories: [linux,git]
tags: [vps, git]
---

在 `~/.ssh/config` 添加 `vps` 的 `ssh` 配置
```txt
Host vps 
    Hostname <vps-ip-address>
    User <vps-user-name>
    Port <vps-port>
    IdentityFile <vps-rsa-file>
```

在 `vps` 上 `fork` github 的 `repo`
```bash
$ git clone --bare https://github.com/apache/arrow.git
Cloning into bare repository 'arrow.git'...
remote: Enumerating objects: 13, done.
remote: Counting objects: 100% (13/13), done.
remote: Compressing objects: 100% (10/10), done.
remote: Total 121519 (delta 3), reused 8 (delta 3), pack-reused 121506
Receiving objects: 100% (121519/121519), 67.04 MiB | 15.94 MiB/s, done.
Resolving deltas: 100% (84048/84048), done.

```

在本地 `clone` `vps` 上 `repo`
```bash
$ git clone vps:~/arrow.git  
Cloning into 'arrow'...
remote: Counting objects: 121519, done.
remote: Compressing objects: 100% (30668/30668), done.
Receiving objects:  42% (51888/121519), 26.00 MiB | 11.00 KiB/s
```

使用 `ssh` clone `vps` 上的 `repo` 可能遇到 `ssh: Could not resolve hostname` 的问题

```bash
 git clone ssh://user@my.host:/path/to/repository
```
上述命令就会遇到这样的问题
```bash
Initialized empty Git repository in /current/path/repository/.git/
ssh: Could not resolve hostname my.host:: Name or service not known
fatal: The remote end hung up unexpectedly
```

解决方案1
```bash
git clone ssh://username@host.xz/absolute/path/to/repo.git/
```

解决方案2
```bash
git clone username@host.xz:relative/path/to/repo.git/
```

参考链接: <https://kuttler.eu/en/post/git-clone-ssh-could-not-resolve-hostname/>

