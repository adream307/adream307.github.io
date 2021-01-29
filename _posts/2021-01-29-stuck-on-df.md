---
title:  "linux 系统 df 命令卡主了"
author: adream307
date:   2021-01-29 14:50:00 +0800
categories: [linux,df]
tags: [df]
---

使用 `df` 命令显示磁盘使用情况，但是卡主了

解决方案： 使用 `strace` 追踪卡在哪里
```bash
$ sudo strace df
stat("/snap/gnome-calculator/748", {st_mode=S_IFDIR|0755, st_size=111, ...}) = 0
stat("/snap/gnome-logs/100", {st_mode=S_IFDIR|0755, st_size=111, ...}) = 0
stat("/cifs/data",
```
发现卡在 `/cifs/data` 目录上, 这是使用 `cifs` 挂着的共享网盘，服务器被移除了



