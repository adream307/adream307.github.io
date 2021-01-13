---
title:  "curl 使用代理访问 google"
author: adream307
date:   2020-11-22 15:24:00 +0800
categories: [linux,curl]
tags: [curl, socks5]
---

`socks5` 代理
```bash
curl -x "socks5h://127.0.0.1:1080" https://www.google.com
```

如果使用 `socks5://`  前缀，而不是 `socks5h://`，那么可以能会得到以下错误。因为 `socks5://` 前缀是在本地解析`www.google.com`，因此可以无法得到正确的 `IP` 地址, 而 `socks5h://`  是在远端解析域名
```bash
$ curl -x "socks5://127.0.0.1:1081" https://www.google.com
curl: (51) SSL: no alternative certificate subject name matches target host name 'www.google.com'
```

`http`  代理
```bash
curl -x "http://127.0.0.1:8123" https://www.google.com
```



