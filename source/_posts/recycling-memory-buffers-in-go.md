---
layout: post
category: default
date: 2013-10-08
title: "复用Go的内存buffer"
description: "复用Go的内存buffer"
tags: [Goc, buffer]
redirecturl: http://blog.jobbole.com/48969/
---


之前我写过我们如何使用Lua[实现我们新的网络应用程序防火墙](http://blog.cloudflare.com/cloudflares-new-waf-compiling-to-lua)。在CloudFlare很流行的另一种语言是[Go](http://golang.org/)。在以前，我已经写过[如何使用Go](http://blog.cloudflare.com/go-at-cloudflare)来写网络服务，如[Railgun](http://blog.cloudflare.com/cacheing-the-uncacheable-cloudflares-railgun-73454)。

使用像Go这样带有垃圾收集机制的语言编写长时间运行网络服务程序，内存管理是一个潜在的挑战。

为了理解Go的内存管理，分析一些Go运行时代码还是有必要的。Go程序中有两个独立的线程，用来标记不再被程序使用的内存（这就是垃圾收集），并在其不再被使用时返还给操作系统（在Go代码中称为收割，scavenging）。

下面是一个小程序，会生成很多内存垃圾，每秒生成一个5MB到10MB的字节数组。它维护了一个20个这样字节数组大小的内存池，随机丢弃内存池中的字节数组。这个程序用来模拟程序中经常发生的场景：程序的各个部分每时每刻都会分配内存，一些分配的内存一直都在使用，大多数分配的内存都不再使用。在一个Go写的网络程序中，在处理网络链接或请求的Go协程里，这种情况很容易发生。常常是这样的，Go协程分配内存块（比如分配一个slices来存储接收的数据），然后就不再使用。随着时间的积累，会有一系列的内存块被正在被处理的网络链接占用，也会有一些累计的来自那些被处理过的链接的内存垃圾。

    package main
     
    import (  
        "fmt"  
        "math/rand"  
        "runtime"  
        "time"
    )
     
    func makeBuffer() []byte {  
        return make([]byte, rand.Intn(5000000)+5000000)  
    }
     
    func main() {  
        pool := make([][]byte, 20)
     
        var m runtime.MemStats  
        makes := 0  
        for {  
            b := makeBuffer()  
            makes += 1
            i := rand.Intn(len(pool))
            pool[i] = b
     
            time.Sleep(time.Second)
     
            bytes := 0
     
            for i := 0; i < len(pool); i++ {
                if pool[i] != nil {
                    bytes += len(pool[i])
                }
            }
     
            runtime.ReadMemStats(&m)
            fmt.Printf("%d,%d,%d,%d,%d,%d\n", m.HeapSys, bytes, m.HeapAlloc,
                m.HeapIdle, m.HeapReleased, makes)
        }
    }

这个程序使用[runtime.ReadMemStats](#ReadMemStats)函数来获取堆大小的信息。这个函数会打印四个值：HeapSys （程序向操作系统请求的内存的字节数），HeapAlloc （当前堆中已经分配的字节数），HeapIdle （堆中未使用的字节数）和HeapReleased （归还给操作系统的字节数）。

Go程序中垃圾收集运行的很频繁（查看GOGC环境变量来理解如何控制GC操作 ）。因此，在运行过程中，堆的大小会随着内存被标记为未使用（这回导致HeapAlloc 和HeapIdle 随之变化）而变化。收割线程只有在内存5分钟都没有使用才会释放内存，因此HeapReleased 并不经常变化。

这是上面的程序运行10分钟生成的图表。

![no1](/post-images/2013-10/no1.png)

(这个图以及下面的所有的图，左边的纵坐标是字节数，右边的纵坐标是Make的次数)

红线表示实际上内存池字节数据占用的字节数。因为内存池大小是20个buffer，所以很快就上涨到150M左右。最上面的蓝线表示程序向操作系统请求的内存数量。最高达到375M左右。所以，程序使用了2.5倍实际需要的内存。

随着程序读写内存，当垃圾收集发生时，堆中分配的大小和未使用的大小上下浮动。桔色的线正好是 makeBuffer()调用的次数。

这类随着请求使用内存在垃圾收集程序中是很常见的（例如，论文[Quantifying the Performance of Garbage Collection vs. Explicit Memory Management](http://cs.canisius.edu/~hertzm/gcmalloc-oopsla-2005.pdf)）。随着程序的运行，堆中未使用的内存又被重新利用，很少会被释放给操作系统。

解决这种问题的一个方法就是在程序中部分地手动管理内存。比如，使用一个管道，可以单独维护一个不再使用字节数组的内存池，当需要新的字节数组时，从内存池中拿（当内存池为空就生成新的字节数组）。

这个程序可以这样重写：

    package main
     
    import (
        "fmt"
        "math/rand"
        "runtime"
        "time"
    )
     
    func makeBuffer() []byte {
        return make([]byte, rand.Intn(5000000)+5000000)
    }
     
    func main() {
        pool := make([][]byte, 20)
     
        buffer := make(chan []byte, 5)
     
        var m runtime.MemStats
        makes := 0
        for {
            var b []byte
            select {
            case b = <-buffer:
            default:
                makes += 1
                b = makeBuffer()
            }
     
            i := rand.Intn(len(pool))
            if pool[i] != nil {
                select {
                case buffer <- pool[i]:
                    pool[i] = nil
                default:
                }
            }
     
            pool[i] = b
     
            time.Sleep(time.Second)
     
            bytes := 0
            for i := 0; i < len(pool); i++ {
                if pool[i] != nil {
                    bytes += len(pool[i])
                }
            }
     
            runtime.ReadMemStats(&m)
            fmt.Printf("%d,%d,%d,%d,%d,%d\n", m.HeapSys, bytes, m.HeapAlloc,
                m.HeapIdle, m.HeapReleased, makes)
        }
    }

下面是新的程序运行10分钟的效果：

![no2](/post-images/2013-10/no2.png)

这张图就和上面的完全不同。内存池中的内存和从操作系统请求的内存很接近。垃圾收集器也基本不做什么。堆中只有很少量的未使用内存最终返还给操作系统。

这种内存复用机制的关键是一个缓存的管道buffer。上面的代码中可以存储5个字节数组。当程序需要一个字节数组时，优先使用select从缓存的管道中去取:

    select {
        case b = <-buffer:
        default:
            b = makeBuffer()
    }

select永远不会阻塞因为如果buffer 管道中有字节数组，第一个分支生效，字节数组赋给了 b。如果管道是空的话（也就意味着receive会阻塞），default 分支会执行，并分配了一个新的字节数组。

把字节数组放回到管道中使用了类似的无阻塞模式:

    select {
        case buffer <- pool[i]:
            pool[i] = nil
        default:
    }

如果buffer 管道已经满了，往管道里面发送就会阻塞。这种情况下，default分支执行，什么也不做。这种简单的机制可以用来安全的生成一个共享的内存池。由于管道通信对多go协程是安全的，这种机制也可以用于go协程的共享。

实际上，我们在Go程序中使用了类似的技术。下面的代码是真实复用器的简化版。使用一个go协程处理字节数组的生成并在软件中共享给所有的go协程。两个管道get （获取一个新的字节数组）和give （返回字节数组到内存池中）在所有的通信中都被使用。

复用器保存了一个返回的字节数组的链表，间断地丢弃那些时间太久，并不再会被复用（示例代码中，生命周期超过1分钟）的字节数组。这使得程序处理对字符数组的动态需求。

    package main
     
    import (
        "container/list"
        "fmt"
        "math/rand"
        "runtime"
        "time"
    )
     
    var makes int
    var frees int
     
    func makeBuffer() []byte {
        makes += 1
        return make([]byte, rand.Intn(5000000)+5000000)
    }
     
    type queued struct {
        when time.Time
        slice []byte
    }
     
    func makeRecycler() (get, give chan []byte) {
        get = make(chan []byte)
        give = make(chan []byte)
     
        go func() {
            q := new(list.List)
            for {
                if q.Len() == 0 {
                    q.PushFront(queued{when: time.Now(), slice: makeBuffer()})
                }
     
                e := q.Front()
     
                timeout := time.NewTimer(time.Minute)
                select {
                case b := <-give:
                    timeout.Stop()
                    q.PushFront(queued{when: time.Now(), slice: b})
     
               case get <- e.Value.(queued).slice:
                   timeout.Stop()
                   q.Remove(e)
     
               case <-timeout.C:
                   e := q.Front()
                   for e != nil {
                       n := e.Next()
                       if time.Since(e.Value.(queued).when) > time.Minute {
                           q.Remove(e)
                           e.Value = nil
                       }
                       e = n
                   }
               }
           }
     
        }()
     
        return
    }
     
    func main() {
        pool := make([][]byte, 20)
     
        get, give := makeRecycler()
     
        var m runtime.MemStats
        for {
            b := <-get
            i := rand.Intn(len(pool))
            if pool[i] != nil {
                give <- pool[i]
            }
     
            pool[i] = b
     
            time.Sleep(time.Second)
     
            bytes := 0
            for i := 0; i < len(pool); i++ {
                if pool[i] != nil {
                    bytes += len(pool[i])
                }
            }
     
            runtime.ReadMemStats(&m)
            fmt.Printf("%d,%d,%d,%d,%d,%d,%d\n", m.HeapSys, bytes, m.HeapAlloc
                 m.HeapIdle, m.HeapReleased, makes, frees)
        }
    }

运行这个程序10分钟效果和第二个程序非常像：

![no3](/post-images/2013-10/no3.png)

这些技术可以在程序员知道内存会被复用而不需要垃圾收集器参与时用来复用内存。它可以显著的减少程序需要内存的大小。并不仅限于字节数组。任何Go类型都可以用类似的行为进行复用。
