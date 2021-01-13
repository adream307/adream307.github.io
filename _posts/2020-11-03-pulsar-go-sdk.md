---
title:  "pulsar go sdk 测试"
author: adream307
date:   2020-11-03 14:45:00 +0800
categories: [pulsar,go-sdk]
tags: [pulsar]
---

## docker 启动 pulsar
```bash
docker run --rm -p 6650:6650 -p 8080:8080 apachepulsar/pulsar:2.6.0 bin/pulsar standalone
```

## pulsar go sdk
```go
package main

import (
	"context"
	"encoding/binary"
	"github.com/apache/pulsar-client-go/pulsar"
	"log"
	"sync"
)

func main() {
	cli, err := pulsar.NewClient(pulsar.ClientOptions{URL: "pulsar://localhost:6650"})
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		i := uint64(i)
		go func() {
			send, err := cli.CreateProducer(pulsar.ProducerOptions{Topic: "hello"})
			if err != nil {
				log.Fatal(err)
			}
			defer send.Close()
			bi := make([]byte, 8)

			binary.LittleEndian.PutUint64(bi, i)
			_, err = send.Send(context.TODO(), &pulsar.ProducerMessage{Payload: bi})
			if err != nil {
				log.Fatal(err)
			}
			log.Printf("send %d into pulsar\n", i)
		}()
	}

	go func() {
		recv, err := cli.Subscribe(pulsar.ConsumerOptions{
			Topic:                       "hello",
			SubscriptionName:            "hello-g",
			Type:                        pulsar.KeyShared,
			SubscriptionInitialPosition: pulsar.SubscriptionPositionEarliest,
		})
		if err != nil {
			log.Fatal(err)
		}
		for i := 0; i < 10; i++ {
			cm, ok := <-recv.Chan()
			if !ok {
				log.Fatalf("pulsar channel closed")
			}
			recv.AckID(cm.ID())
			val := binary.LittleEndian.Uint64(cm.Payload())
			log.Printf("recv %d from pulsar\n", val)
			wg.Done()
		}
	}()

	wg.Wait()
	log.Printf("finish")
}
```