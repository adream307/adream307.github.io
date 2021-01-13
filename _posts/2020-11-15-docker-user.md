---
title:  "docker 中创建新用户"
author: adream307
date:   2020-11-15 21:05:00 +0800
categories: [linux,docker]
tags: [docker]
---

`Dockerfile`
```dockerfile
FROM ubuntu:18.04

RUN sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
RUN sed -i 's/security.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
RUN useradd -m test123 -s /bin/bash
RUN echo 'test123\ntest123' | passwd test123
```

上述 `Dockerfile` 创建一个 `test123` 的用户，并设置其密码为 `test123`

创建 `docker image`
```bash
#!/bin/bash
docker build -t test123:1804 .
```
