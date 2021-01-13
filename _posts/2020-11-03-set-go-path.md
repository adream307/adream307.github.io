---
title:  "bash 脚本设置 GOPATH"
author: adream307
date:   2020-11-03 19:25:00 +0800
categories: [linux,bash]
tags: [bash]
---

## go-path.sh
```bash
#!/bin/bash

if [ $# -ne 1 ];then
        echo "usage : go-path: <go-path>"
        exit 0
fi

if [ ! -z $GOPATH ];then
        echo "origin GOPATH = ${GOPATH}"
        echo "origin PATH = ${PATH}"
        go_bin_path=:${GOPATH}/bin
        if [[ $PATH == *${go_bin_path}* ]];then
                PATH=${PATH/${go_bin_path}}
        fi
fi

export GOPATH=$1
if [[ $PATH != *${GOPATH}* ]];then
        export PATH=$PATH:$GOPATH/bin
fi

echo "GOPATH = ${GOPATH}"
echo "PATH = ${PATH}"

export http_proxy="http://127.0.0.1:8123"
export https_proxy="http://127.0.0.1:8123"

```