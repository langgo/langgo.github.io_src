---
title: go 并发模式
date: 2018-10-16 15:42:24
tags:
---

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func Start(ctx context.Context) <-chan int {
	ch := make(chan int, 64)
	go func() {
		defer func() {
			close(ch)
		}()

		i := 0
		for {
			select {
			case <-ctx.Done():
				return
			case <-time.After(1 * time.Second):
				ch <- i
				i++
			}
		}
	}()
	return ch
}

func Exec(ch <-chan int) error {
	var wg sync.WaitGroup
	for i := 0; i < 4; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()

			v := <-ch
			fmt.Println(i, v)
		}(i)
	}
	wg.Wait()

	// 执行资源清理操作
	return nil
}

// Start
// Exec
// 每个函数都是同步函数
// 遵循谁 send 谁 close
// 谁使用谁清理

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	time.AfterFunc(5*time.Second, func() {
		cancel()
	})

	ch := Start(ctx)
	if err := Exec(ch); err != nil {
		panic(err)
	}
}
```