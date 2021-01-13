---
title:  "zap配合logrotate实现日志滚动"
author: adream307
date:   2020-09-10 17:0:00 +0800
categories: [linux, logrotate]
tags: [zap, logrotate, go,]
---

[zap](https://github.com/uber-go/zap)是`Uber` 提供的`GoLang`高性能日志库，`zap` 本身并不提供日志滚动功能，官方 `FAQ` 提到，可以使用`Linux`系统自带的 `logrotate` 或`lumberjack`实现日志滚动功能

`lumberjack` 只能向文件输出日志，如果我们希望同时向`stderr` 和文件输出日志，只能使用 `logrotate` 配合自定义 `WriteSyncer` 实现了


## `Go`代码
```go
package main

import (
	"bufio"
	"context"
	"fmt"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"os"
	"os/signal"
	"sync"
	"syscall"
)

type LogWriter struct {
	FileName string
	fp       *os.File
	mu       sync.Mutex
	fileCnt  int
	sig      chan os.Signal
	ctx      context.Context
	cancel   context.CancelFunc
}

func (m *LogWriter) Write(p []byte) (n int, err error) {
	m.mu.Lock()
	defer m.mu.Unlock()
	if m.fp == nil {
		if err := m.open(); err != nil {
			return 0, nil
		}
	}
	if m.fileCnt == 0 {
		m.fileCnt++
		m.ctx, m.cancel = context.WithCancel(context.Background())
		m.sig = make(chan os.Signal, 1)
		signal.Notify(m.sig, syscall.SIGHUP)
		go func() {
			for {
				select {
				case <-m.ctx.Done():
					return
				case _, ok := <-m.sig:
					if ok == false {
						return
					}
					func() {
						m.mu.Lock()
						defer m.mu.Unlock()
						if m.fp != nil {
							_ = m.fp.Close()
							m.fp = nil
						}
					}()
				}
			}
		}()
	}
	_, _ = os.Stderr.Write(p)
	return m.fp.Write(p)
}

func (m *LogWriter) Sync() error {
	m.mu.Lock()
	defer m.mu.Unlock()
	_ = os.Stderr.Sync()
	return m.fp.Sync()
}

func (m *LogWriter) Close() error {
	m.mu.Lock()
	defer m.mu.Unlock()
	m.cancel()
	if m.fp != nil {
		if err := m.fp.Close(); err != nil {
			return err
		}
	}
	if m.sig != nil {
		signal.Reset(syscall.SIGHUP)
		close(m.sig)
	}
	return nil
}

func (m *LogWriter) open() error {
	if m.fp != nil {
		if err := m.fp.Sync(); err != nil {
			return err
		}
		if err := m.fp.Close(); err != nil {
			return err
		}
	}
	fp, err := os.OpenFile(m.FileName, os.O_APPEND|os.O_WRONLY|os.O_CREATE, 0644)
	if err != nil {
		return err
	}
	m.fp = fp
	return nil
}

func main() {
	zapid, _ := os.OpenFile("/tmp/zap.pid", os.O_CREATE|os.O_WRONLY, 0644)
	_, _ = zapid.WriteString(fmt.Sprintf("%d", os.Getpid()))
	_ = zapid.Sync()
	_ = syscall.Flock(int(zapid.Fd()), syscall.LOCK_EX)

	mylog := &LogWriter{FileName: "/tmp/zap.log"}
	cfg := zap.NewProductionEncoderConfig()
	cfg.EncodeTime = zapcore.ISO8601TimeEncoder
	core := zapcore.NewCore(zapcore.NewJSONEncoder(cfg), mylog, zap.InfoLevel)
	logger := zap.New(core)

	reader := bufio.NewReader(os.Stdin)
	for i := 0; i < 10; i++ {
		logger.Debug("debug msg", zap.Int("idx", i))
		logger.Info("info msg", zap.Int("idx", i))
		logger.Error("error msg", zap.Int("idx", i))
		logger.Warn("warn msg", zap.Int("idx", i))
		fmt.Printf("====================")
		_, _, _ = reader.ReadRune()
	}
	_ = mylog.Close()

	_ = syscall.Flock(int(zapid.Fd()), syscall.LOCK_UN)
	_ = zapid.Close()
}

```
编译生成测试程序 `zap_exp`
```bash
go build zap_exp.go
```

## `logrotate` 配置文件
```conf
/tmp/zap.log {
    su robin robin
    missingok
    daily
    minsize 1M 
    create 0644 robin robin
    rotate 12
    postrotate
        PIDFILE=/tmp/zap.pid
        if [ -f "$PIDFILE" ] ; then kill -HUP $(cat "$PIDFILE") ; fi
    endscript
}
```
`zap.conf` 的 `owner` 和 `group` 必须为 `root`，并且 `file mode` 为 `0644` ，只有 `root` 可以读写，其它只读
- `su robin robin`  : 运行 `zap_exp` 程序的用户名及坐在的 `Group`，本测试中 `user` 和 `group` 均为 `robin`
- ` create 0644 robin robin` : 日志滚动后，重新生成日志文件的 `file mode` 及 `ower` 和 `group`
-  `postrotate` : 需要配合`zap_exp.go` 源码分析
	- `zap_exp.go`  在 `main` 函数启动后首先生成 `/tmp/zap.pid`文件，然后将当前程序的 `pid` 写入该文件，并锁定该文件
	- `LogWriter` 设置 `signal.Notify(m.sig, syscall.SIGHUP)` 表明 `zap_exp` 程序接受 `HUP` 信号
	- `LogWriter` 收到 `HUP` 信号后关闭当前日志文件
	- `LogWriter`写日志文件时，如果日志文件被关闭，程序自动重新创建
	- `logrotate` 程序运行时，首先从`/tmp/zap.pid`文件读取`zap_exp` 程序的 `pid` ，然后向`zap_exp` 程序发送 `HUP` 信号
 

## 测试
运行 `zap_exp` 程序，分别在 `idx:3` 和 `idx:6` 时手动强制执行 `logrotate` 程序
```bash
logrotate -s ./log.status -vf ./zap.conf
```
`zap_exp` 运行结果
```bash
./zap_exp
{"level":"info","ts":"2020-09-10T16:53:54.896+0800","msg":"info msg","idx":0}
{"level":"error","ts":"2020-09-10T16:53:54.896+0800","msg":"error msg","idx":0}
{"level":"warn","ts":"2020-09-10T16:53:54.896+0800","msg":"warn msg","idx":0}
====================
{"level":"info","ts":"2020-09-10T16:53:56.628+0800","msg":"info msg","idx":1}
{"level":"error","ts":"2020-09-10T16:53:56.628+0800","msg":"error msg","idx":1}
{"level":"warn","ts":"2020-09-10T16:53:56.628+0800","msg":"warn msg","idx":1}
====================
{"level":"info","ts":"2020-09-10T16:53:57.060+0800","msg":"info msg","idx":2}
{"level":"error","ts":"2020-09-10T16:53:57.060+0800","msg":"error msg","idx":2}
{"level":"warn","ts":"2020-09-10T16:53:57.060+0800","msg":"warn msg","idx":2}
====================
{"level":"info","ts":"2020-09-10T16:53:57.393+0800","msg":"info msg","idx":3}
{"level":"error","ts":"2020-09-10T16:53:57.393+0800","msg":"error msg","idx":3}
{"level":"warn","ts":"2020-09-10T16:53:57.393+0800","msg":"warn msg","idx":3}
====================
{"level":"info","ts":"2020-09-10T16:54:31.706+0800","msg":"info msg","idx":4}
{"level":"error","ts":"2020-09-10T16:54:31.706+0800","msg":"error msg","idx":4}
{"level":"warn","ts":"2020-09-10T16:54:31.706+0800","msg":"warn msg","idx":4}
====================
{"level":"info","ts":"2020-09-10T16:54:32.050+0800","msg":"info msg","idx":5}
{"level":"error","ts":"2020-09-10T16:54:32.050+0800","msg":"error msg","idx":5}
{"level":"warn","ts":"2020-09-10T16:54:32.050+0800","msg":"warn msg","idx":5}
====================
{"level":"info","ts":"2020-09-10T16:54:32.359+0800","msg":"info msg","idx":6}
{"level":"error","ts":"2020-09-10T16:54:32.359+0800","msg":"error msg","idx":6}
{"level":"warn","ts":"2020-09-10T16:54:32.359+0800","msg":"warn msg","idx":6}
====================
{"level":"info","ts":"2020-09-10T16:54:36.955+0800","msg":"info msg","idx":7}
{"level":"error","ts":"2020-09-10T16:54:36.956+0800","msg":"error msg","idx":7}
{"level":"warn","ts":"2020-09-10T16:54:36.956+0800","msg":"warn msg","idx":7}
====================
{"level":"info","ts":"2020-09-10T16:54:37.255+0800","msg":"info msg","idx":8}
{"level":"error","ts":"2020-09-10T16:54:37.255+0800","msg":"error msg","idx":8}
{"level":"warn","ts":"2020-09-10T16:54:37.255+0800","msg":"warn msg","idx":8}
====================
{"level":"info","ts":"2020-09-10T16:54:37.712+0800","msg":"info msg","idx":9}
{"level":"error","ts":"2020-09-10T16:54:37.712+0800","msg":"error msg","idx":9}
{"level":"warn","ts":"2020-09-10T16:54:37.712+0800","msg":"warn msg","idx":9}
====================
```
在 `/tmp` 可以找到如下`3`个文件 `zap.log ` `zap.log.1` `zap.log.2`
`zap.log.2` 内容如下，刚好对应 `idx:3` 时执行 `logrotate` 时产生的日志滚动
```txt
{"level":"info","ts":"2020-09-10T16:53:54.896+0800","msg":"info msg","idx":0}
{"level":"error","ts":"2020-09-10T16:53:54.896+0800","msg":"error msg","idx":0}
{"level":"warn","ts":"2020-09-10T16:53:54.896+0800","msg":"warn msg","idx":0}
{"level":"info","ts":"2020-09-10T16:53:56.628+0800","msg":"info msg","idx":1}
{"level":"error","ts":"2020-09-10T16:53:56.628+0800","msg":"error msg","idx":1}
{"level":"warn","ts":"2020-09-10T16:53:56.628+0800","msg":"warn msg","idx":1}
{"level":"info","ts":"2020-09-10T16:53:57.060+0800","msg":"info msg","idx":2}
{"level":"error","ts":"2020-09-10T16:53:57.060+0800","msg":"error msg","idx":2}
{"level":"warn","ts":"2020-09-10T16:53:57.060+0800","msg":"warn msg","idx":2}
{"level":"info","ts":"2020-09-10T16:53:57.393+0800","msg":"info msg","idx":3}
{"level":"error","ts":"2020-09-10T16:53:57.393+0800","msg":"error msg","idx":3}
{"level":"warn","ts":"2020-09-10T16:53:57.393+0800","msg":"warn msg","idx":3}
```


## 参考资料
- [zap FAQ](https://github.com/uber-go/zap/blob/master/FAQ.md#does-zap-support-log-rotation)
- [zap github issue : use with linux logrotate ](https://github.com/uber-go/zap/issues/797)

