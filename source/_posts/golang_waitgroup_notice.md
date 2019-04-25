---
title: golang中WaitGroup的一点坑
date: 2019-04-22 10:04:39
tags: [go, waitgroup]
categories: [develop]
---

## Golang中WaitGroup使用的一点坑
Golang 中的 WaitGroup 一直是同步 goroutine 的推荐实践。

## 坑1
``` go
package main

import (
  "log"
  "sync"
)

func main() {
  wg := sync.WaitGroup{}

  for i := 0; i &lt; 5; i++ {
    go func(wg sync.WaitGroup, i int) {
      wg.Add(1)
      log.Printf("i:%d", i)
      wg.Done()
    }(wg, i)
  }

  wg.Wait()

  log.Println("exit")
}
```

<!-- more -->

运行结果是这样：
```
test@VM-PC:/tmp$ ./demo
2019/04/22 10:05:38 exit
test@VM-PC:/tmp$ ./demo
2019/04/22 10:05:40 i:3
2019/04/22 10:05:40 exit
test@VM-PC:/tmp$ ./demo
2019/04/22 10:05:41 i:4
2019/04/22 10:05:41 i:0
2019/04/22 10:05:41 exit
```
如果理解了WaitGroup的设计目的就非常容易理解这个问题啦。因为WaitGroup同步的是goroutine,  
而上面却在goroutine中进行Add(1)操作。因此可能goroutine还没来得及Add(1)已经执行Wait操作了。 
于是代码改成了这样：

## 坑2
``` go
package main

import (
  "log"
  "sync"
)

func main() {
  wg := sync.WaitGroup{}

  for i := 0; i &lt; 5; i++ {
    wg.Add(1)
    go func(wg sync.WaitGroup, i int) {
      log.Printf("i:%d", i)
      wg.Done()
    }(wg, i)
  }

  wg.Wait()

  log.Println("exit")
}
```

运行结果是这样：
```
test@VM-PC:/tmp$ ./demo
2019/04/22 10:05:59 i:4
2019/04/22 10:05:59 i:1
2019/04/22 10:05:59 i:0
2019/04/22 10:05:59 i:2
2019/04/22 10:05:59 i:3
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
sync.runtime_Semacquire(0xc42008a02c)
    /usr/lib/go-1.10/src/runtime/sema.go:56 +0x39
    sync.(*WaitGroup).Wait(0xc42008a020)
    /usr/lib/go-1.10/src/sync/waitgroup.go:129 +0x72
    main.main()
    /tmp/demo.go:20 +0xa2
```
wg给拷贝传递到了goroutine中，导致只有Add操作，其实Done操作是在wg的`副本`执行的。  
因此Wait就死锁了。于是代码改成了这样：

## 填坑
``` go
package main

import (
  "log"
  "sync"
)

func main() {
  wg := sync.WaitGroup{}

  for i := 0; i &lt; 5; i++ {
    wg.Add(1)
    go func(wg *sync.WaitGroup, i int) {
      log.Printf("i:%d", i)
      wg.Done()
    }(&wg, i)
  }

  wg.Wait()

  log.Println("exit")
}
```
运行结果是这样：
```
test@VM-PC:/tmp$ ./demo
2019/04/22 10:06:42 i:4
2019/04/22 10:06:42 i:1
2019/04/22 10:06:42 i:0
2019/04/22 10:06:42 i:2
2019/04/22 10:06:42 i:3
2019/04/22 10:06:42 exit
```

