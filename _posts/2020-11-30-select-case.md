---
title:  "go 语言动态添加 select case"
author: adream307
date:   2020-11-30 13:25:00 +0800
categories: [linux,go]
tags: [go]
---
`go` 语言中传统的 `select case` 必须固定写死，即我们在编码阶段必须明确知道当前有几个 `case`,如下
```go
select{
    case <- chan1:
        //todo
    case <- chan2:
        //todo
    case <- chan3:
        //todo
    case <- chan4:
        //todo
}
```
如果我在编码是不确定有几个 `case`，只在运行是才能知道，应该如何处理? 示例代码如下
```go
package main

import (
	"context"
	"log"
	"math/rand"
	"reflect"
	"sync"
)

const (
	numChannel int = 10
)

func main() {
	cases := make([]reflect.SelectCase, 0, numChannel+1)
	chans := make([]chan int, 0, numChannel)
	for i := 0; i < numChannel; i++ {
		ch := make(chan int)
		chans = append(chans, ch)
		ch_case := reflect.SelectCase{
			Dir:  reflect.SelectRecv,
			Chan: reflect.ValueOf(ch),
		}
		cases = append(cases, ch_case)
	}
	ctx, cancel := context.WithCancel(context.TODO())
	cases = append(cases, reflect.SelectCase{
		Dir:  reflect.SelectRecv,
		Chan: reflect.ValueOf(ctx.Done()),
	})

	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		for {
			chosen, val, ok := reflect.Select(cases)
			if !ok {
				if chosen == numChannel {
					log.Print("context cancel")
				} else {
					log.Printf("channel %d closed", chosen)
				}
				return
			}
			intVal, ok := val.Interface().(int)
			if !ok {
				log.Print("unexpect data type")
				return
			}
			log.Printf("channel %d receive value %d", chosen, intVal)
		}
	}()

	for i:=0;i<1000;i++{
		idx := rand.Int()%numChannel
		chans[idx] <- rand.Int()
	}
	cancel()
	wg.Wait()
}

```
