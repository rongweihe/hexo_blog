---
title: Golang 并发编程实战总结
date: 2025-03-28 16:12:34
tags: golang
categories: 博客
---

# Golang 并发编程实践总结

目录
- [Golang 并发编程实践总结](#golang-并发编程实践总结)
  - [深⼊理解GMP模型](#深⼊理解GMP模型)
  - [Channel是什么](#Channel是什么？什么作用？底层数据结构是什么？)
  - [Select的用法](#Select的用法)
  - [使用sync包](#使用sync包)
  - [WaitGroup](#WaitGroup)
  - [Pool](#Pool)
  - [常见内存泄露及注意事项](#常见内存泄露及注意事项)

# 深⼊理解GMP模型

**【思考🤔】⾸先思考⼀个问题为什么需要协程？**

其实答案也很明显：在多进程和多线程时代，CPU 内核负责调度，为系统提供并发处理能⼒，但也存在⼀些缺点：

- 资源消耗⾼：进程和线程的创建、切换和销毁都会消耗⼤量 CPU 资源。

- 内存占⽤⼤：每个线程约需 4MB 内存，⼤量线程会导致内存消耗过⾼。

- 应⽤层⽆法直接控制内核调度，只能通过减少线程创建和切换来优化性能。

这促⽣了协程的概念：⽤户级别的轻量线程。在 Go 中，协程被称为 `goroutine`，它主要解决了内核线程的两个“太重”问题：

- 创建和切换：`goroutine`  在⽤户态创建和切换，⽆需进⼊内核，开销⽐线程⼩得多。
- 内存占⽤：`goroutine`  的初始栈只有 2KB，栈空间可动态扩展或收缩，避免了内存浪费与栈溢出⻛险。
- 由于 `goroutine`  的轻量特性，Go 程序可以轻松创建成千上万个并发任务，⽽不⽤担⼼性能和内存问题。



**【思考🤔 】第⼆个问题：**golang **可以在⽤户级别空间创建协程底层是靠什么实现的**?

其实是虽然协程在⽤户态空间运⾏，但其底层实现依赖于Go运⾏时对操作系统线程（OS Threads）的管理和调度。

具体来说：Go 协程运⾏在⽤户态空间，但它们并不是直接由操作系统调度的线程。相反，**Go **运⾏时通过⼀个⽤户态的调度器（**Scheduler**）来管理协程的执⾏，并将协程映射到操作系统线程上。

- ⽤户态协程（Goroutine）：轻量级的并发单元，由Go运⾏时管理。
- 操作系统线程（OS Thread）：由操作系统调度的线程，⽤于执⾏Go运⾏时的代码。
- 运⾏时调度器（Scheduler）：负责将协程分配到操作系统线程上执⾏。

Go 运⾏时的调度器是协程实现的核⼼，它通过以下组件协同⼯作：

- G（goroutine）：
  - 是⽤户态线程的抽象，可以在 M 上运⾏，存储于全局队列和 P 的本地队列（⼤⼩ 256）中。
  - **特点：**G是轻量级的，创建和销毁的开销极⼩。
- M（Machine）：
  - 是操作系统线程的抽象，⼀个 M 代表⼀个线程，最多绑定⼀个 P。M 阻塞时会释放 P，允许 P与其他空闲的 M 绑定，如果没有空闲的 M，则创建新的 M。
  - **特点：**M的数量通常远少于G的数量，由Go运⾏时动态管理。
- P（Processor）：
  - 逻辑处理器，抽象代表 CPU 核⼼。P 的数量决定了程序的并⾏能⼒，可以通过 GOMAXPROCS 设置。每个 M 需要绑定⼀个 P 进⾏任务调度。是 M 和 G 之间的桥梁。每个 P 都有⼀个本地的 Goroutine 队列，同时也可以访问全局的 Goroutine 队列。P 负责调度和管理 Goroutine 的执⾏，决定哪个 Goroutine 应该在哪个 M 上运⾏。
  - **特点：** P的数量通常与CPU核⼼数相匹配，⽤于充分利⽤多核CPU的计算能⼒。

Go运⾏时的调度器通过以下步骤管理协程的执⾏：

- 创建协程：当调⽤go func()时，Go运⾏时会创建⼀个新的G，并将其加⼊到调度队列中。
- 调度协程：调度器从队列中取出G，并将其分配给⼀个P。P将G绑定到⼀个M上，M开始执⾏G中的代码。

阻塞与唤醒

- 如果G在执⾏过程中阻塞（如等待I/O操作），调度器会将该G挂起，并从队列中取出另⼀个G继续执⾏。
- 当阻塞的G被唤醒时，调度器会将其重新加⼊到队列中，等待执⾏。

线程复⽤：

- 如果所有M都被阻塞，调度器会创建新的M来执⾏队列中的G。
- 如果某个M完成任务，调度器会将该M回收，⽤于执⾏其他G。

GMP 模型引⼊了 P 来解决 GM 模型的缺陷：

- ⽆锁访问本地队列：P 保存 goroutine 的本地队列，M 优先从 P 的本地队列中取 G 执⾏，仅必要时访问全局队列，减少了锁竞争，提升了并发性。
- 优化数据局部性：新创建的 G 会优先放⼊创建它的 M 绑定的 P 的本地队列中，避免频繁的数据交互，提升内存使⽤效率。
- Hand Off 交接机制：M 阻塞时，将绑定的 P 和其任务交给其他 M，提升了资源利⽤率和并发度。

GMP 调度时机：介绍 GMP 的调度时机，主要分为正常调度、主动调度、被动调度和抢占调度四种情况

具体调度过程

- **创建** Goroutine：当使⽤ go 关键字创建⼀个新的 Goroutine 时，Go 运⾏时会为其分配⼀个 G 对象，并将其放⼊全局或某个 P 的本地 Goroutine 队列中。

- **调度** Goroutine **到** M **上执⾏**：M 会从 P 的本地队列或全局队列中获取⼀个 G，并将其绑定到⾃⼰身上执⾏。如果 P 的本地队列为空，M 会尝试从其他 P 的本地队列中 “偷取” ⼀些 G 来执⾏，这种机制称为 “⼯作窃取”，可以提⾼调度的效率。

- **阻塞和唤醒**：当⼀个 Goroutine 遇到阻塞操作（如 I/O 操作、Channel 读写等）时，M 会将该 G 标记为阻塞状**阻塞和唤醒**：当⼀个 Goroutine 遇到阻塞操作（如 I/O 操作、Channel 读写等）时，M 会将该 G 标记为阻塞状态，并将其从当前 P 的队列中移除。同时，M 会继续从队列中获取其他 G 来执⾏。当阻塞操作完成后，G 会被唤醒，并重新放⼊某个 P 的队列中等待调度。

- **上下⽂切换**：Goroutine 的上下⽂切换由 Go 运⾏时负责，⽽不是操作系统内核。这种⽤户态的上下⽂切换⽐内核态的上下⽂切换开销⼩得多，因此可以实现⾼效的并发调度。



# Channel是什么？什么作用？底层数据结构是什么？

## Channel 的作用

在 Golang 并发编程中，channel（通道）是一种用于 goroutine 之间通信和同步的机制。它的主要作用包括：

1. **goroutine 间通信**：channel 允许不同的 goroutine 之间传递数据，实现数据的共享和交换。
2. **goroutine 同步**：通过 channel 可以控制 goroutine 的执行顺序，实现对并发操作的同步。

## Channel 的底层原理

```go
type hchan struct {

  qcount uint // 当前缓冲区中的元素数量

  dataqsiz buf uint unsafe.Pointer // 缓冲区⼤⼩（⽆缓冲区时为 0）

  // 指向缓冲区的指针

  elemsize uint16 // 元素⼤⼩

  closed uint32 // 标记通道是否已关闭

  sendx uint // 下⼀个发送操作的索引

  recvx uint // 下⼀个接收操作的索引

  recvq waitq // 等待接收的协程队列

  sendq waitq // 等待发送的协程队列

  lock mutex // 互斥锁，⽤于保护并发操作
}


```

buf 指向底层循环数组，只有缓冲型的 channel 才有。⽆缓冲区的通道则直接在发送和接收操作之间传递数据。

 sendx， recvx 均指向底层循环数组，表示当前可以发送和接收的元素位置索引值（相对于底层数组）。

 sendq， recvq 分别表示被阻塞的 goroutine，这些 goroutine 由于尝试读取 channel 或向 channel 发送数据⽽被阻

  塞。

waitq 是 sudog 的⼀个双向链表，⽽ sudog 实际上是对 goroutine 的⼀个封装：

```go
type waitq struct {

  first *sudog

  last *sudog

}
```

lock ⽤来保证每个读 channel 或写 channel 的操作都是原⼦的。

- **⽆缓冲区通道**： dataqsiz 为 0，发送和接收操作必须同步完成。如果发送⽅没有接收⽅，发送⽅会阻塞，反之

亦然。

- **有缓冲区通道**： dataqsiz ⼤于 0，发送⽅可以将数据写⼊缓冲区，接收⽅可以从缓冲区读取数据。当缓冲区满

时，发送⽅阻塞；当缓冲区为空时，接收⽅阻塞。

hchan 使⽤互斥锁（ mutex）来保护对缓冲区和协程队列的访问，确保在并发操作时不会出现数据竞争。通过这些底层结构和机制， channel 实现了⾼效且安全的协程间通信。

在 Golang 中，channel 可以分为无缓冲 channel 和有缓冲 channel，它们的主要区别在于通信方式和使用场景。以下是两者的通俗理解和具体区别：

## 通俗理解无缓冲和有缓冲

**无缓冲 channel（同步 channel）**

- 无缓冲 channel 就像两个人直接进行面对面的交流。
- 发送方和接收方必须同时准备好，才能进行数据传递。
- 如果发送方先到，它会一直等待接收方到来；如果接收方先到，它会一直等待发送方发送数据。
- 无缓冲 channel 的发送和接收操作是同步的，会阻塞直到双方都准备好。

**有缓冲 channel（异步 channel）**

- 有缓冲 channel 就像一个快递柜，发送方可以先把东西放在快递柜里，接收方可以在有空的时候来取。
- 发送方在缓冲区有空位时可以直接把数据放进去，不需要等待接收方。
- 接收方在缓冲区有数据时可以直接取出来，不需要等待发送方。
- 有缓冲 channel 的发送和接收操作在缓冲区有空间或数据时可以异步进行，只有在缓冲区满或空时才会阻塞。

使用场景

**无缓冲 channel**

- 当需要严格同步两个 goroutine 的执行顺序时。
- 当需要确保发送方和接收方同时参与通信时。

**有缓冲 channel**

- 当需要解耦发送和接收操作时。
- 当需要暂存数据，避免发送方或接收方阻塞时。



# Select的用法

Select 用来处理一个 Goroutine 等待多个 Channel 的情况。类似于 Switch 语句，Select 的每个 case 都是一个通信操作（发送或接收），当有 case 满足条件的时候，则执行对应 case 块的代码。

当没有 case 满足的时候，Select 逻辑块也会阻塞程序，当有多个 case 满足的时候，Select 会公平选择一个满足条件的 case 语句块执行（GO1.8 之后会随机选择一个）。为了避免阻塞，可以使用 default 。

在 Golang 并发编程中，`select` 语句用于处理多个 channel 的通信。它允许程序同时监听多个 channel 上的通信操作，并根据哪个操作最先完成来执行相应的代码块。`select` 语句在处理多个并发通信时非常有用，可以实现多路复用。

## 关键点

1. **多路复用**：`select` 会同时监听多个 channel 上的通信操作。
2. **随机选择**：如果有多个通信操作同时可以执行，`select` 会随机选择一个执行。
3. **阻塞**：如果没有通信操作可以立即执行且没有默认分支，`select` 会阻塞，直到至少有一个通信操作可以执行。
4. **默认分支**：如果有默认分支，当没有通信操作可以立即执行时，会执行默认分支的代码。

## 注意事项

1. **随机选择**：当多个 `case` 同时满足时，`select` 会随机选择一个执行，无法预测具体是哪一个。
2. **阻塞**：如果没有 `default` 分支且没有通信操作可以立即执行，`select` 会阻塞。
3. **死锁风险**：如果所有 `case` 都无法执行且没有 `default` 分支，程序可能会陷入死锁。
4. **效率**：`select` 的效率较高，适合处理多个并发通信。





# 使用sync包

Golang 并发编程离不开 sync 包，里面提供了很多有用的类型和方法。常用知识点如下：

## Map

普通的 map 不是并发安全的，容易出现竞争问题。sync.Map 是官方提供的一个并发安全的 Map，不需要自己额外处理锁、协调等操作。该类型提供了以下方法：

- Loads(): 装载键值（获取键值）
- Store(): 储存键值
- Delete(): 删除键值
- LoadOrStore(): 如果存在则获取键值，不存在则存储
- Range(): 遍历 Map



## 相反，并发安全的数据有如下

- sync.Map：通过读写 map来实现，读请求默认⾛读 map，如果读不到再请求写 map（加锁保证），同时在特定条件满⾜的情况下同步缓存和 DB 中的数据。misses 字段则⽤于计数，记录缓存未命中的次数，当我们要读取map 中某个 key 对应的值时，优先从 read 读取，如果 read 中不存在，则 misses 值加 1，然后继续从 dirty 中读取，当 misses 值达到某个阈值时，sync.Map 就会将 dirty 提升为 read。

- sync.Pool是⼀个并发安全的临时对象池。它可以安全地被多个 goroutine 同时使⽤，⽤于缓存和复⽤临时对象，减少频繁创建和销毁对象带来的内存分配和垃圾回收压⼒，提⾼性能。
- 原⼦类型包 sync/atomic 包;进⾏并发安全的操作。
- 读写锁（ sync.RWMutex）包装的数据结构。

- Mutex：互斥锁。对于加了锁的操作，其他的 Goroutine 在读取相应对象时会阻塞，知道互斥对象释放。频繁、大量的使用互斥锁会导致性能的降低，应该只在必须用的地方用。互斥锁使用不当还有可能造成死锁、活锁、饿死等情况，就比较复杂了。

## `sync.Once`

在 Golang 中，`sync.Once` 是一个用于确保某个函数或方法只被调用一次的工具。它在并发编程中非常有用，可以用来实现单例模式、资源初始化等场景，确保某些操作只执行一次，即使有多个 goroutine 同时尝试执行这些操作。

### 基本用法

`sync.Once` 提供了两个主要方法：

1. **`Do(f func())`**：执行给定的函数 `f`，并且保证 `f` 只会被执行一次，即使有多个 goroutine 调用 `Do` 方法。
2. **`Done() chan struct{}`**：返回一个通道，当 `Do` 方法完成时，该通道会关闭。可以用于等待 `Do` 方法的完成。

示例代码

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var once sync.Once

    // 定义一个需要只执行一次的函数
    initFunc := func() {
        fmt.Println("Initializing...")
    }

    // 多个 goroutine 同时调用 once.Do
    for i := 0; i < 5; i++ {
        go func() {
            once.Do(initFunc)
        }()
    }

    // 等待所有 goroutine 完成
    time.Sleep(100 * time.Millisecond)
}
```

单例模式

```go
package main

import (
    "fmt"
    "sync"
)

var (
    instance *MySingleton
    once     sync.Once
)

type MySingleton struct{}

func NewMySingleton() *MySingleton {
    once.Do(func() {
        instance = &MySingleton{}
        fmt.Println("Singleton instance created")
    })
    return instance
}

func main() {
    // 多个 goroutine 同时获取单例实例
    for i := 0; i < 5; i++ {
        go func() {
            singleton := NewMySingleton()
            fmt.Println("Got singleton instance:", singleton)
        }()
    }

    // 等待所有 goroutine 完成
    time.Sleep(100 * time.Millisecond)
}
```

输出：

```go
Singleton instance created
Got singleton instance: &{{}}
Got singleton instance: &{{}}
Got singleton instance: &{{}}
Got singleton instance: &{{}}
Got singleton instance: &{{}}
```

### 注意事项

1. **线程安全性**：`sync.Once` 是线程安全的，可以安全地在多个 goroutine 中使用。
2. **函数执行**：`Do` 方法中的函数只会被执行一次，即使有多个 goroutine 同时调用 `Do` 方法。
3. **不可重置**：`sync.Once` 不能被重置，一旦执行完成，无法再次执行。

`sync.Once` 是 Golang 并发编程中一个非常有用的工具，用于确保某个函数或方法只被调用一次。它在实现单例模式、资源初始化等场景中非常有用，可以简化代码并提高程序的可靠性。



## 前言

最近的项目都使用 Golang 进行开发，因为自己是从头自学，难免踩了很多坑。目前项目接近尾声，抽空对一些重要的知识点做一些总结。

## Goroutine 与 Channel

什么是 Goroutine 呢？有人说它是轻量级的进程，或者干脆说是协程。其实都不太准确，协程仅仅是在一个进程中进行子程序的切换，而 Goroutine 是可以多线程多路复用的。简而言之，Goroutine 是一个轻量级的、与其他 Goroutine 运行在同一块内存地址中的并发执行的函数模型。它的成本仅仅比堆栈的分配高一点点，所以很廉价。Goroutine 可以在多个线程上通过 Goroutine 自己的调度器实现多路复用。

那什么是 Channel 呢？它是程序中一种类型化的通道，即在这个虚拟通道中传输的是某种预定义的数据类型。通过相应的操作符如 `<-` 可以在 Goroutine 之间发送和接收数据。 Channel 在默认情况下，接收或者发送的某端没有执行的时候会阻塞程序。Channel 可以在创建时定义缓冲大小，即缓冲区已满时才会发送缓存区的所有数据。

Channel 最简单的用法就是利用其阻塞程序的特性来做 Goroutine 结束的标志。例如在 Goroutine 代码块外定义一个 Channel 接收其中的数据，当 Goroutine 代码块内部执行完毕时向 Channel 发送完成信号进而执行后续的代码。

当程序有多个 Goroutine 或者多个 Channel ，他们管理起来就容易变得混乱。这时就可以考虑使用下文讲到的知识。

## Select 的用法

Select 用来处理一个 Goroutine 等待多个 Channel 的情况。类似于 Switch 语句，Select 的每个 case 都是一个通信操作（发送或接收），当有 case 满足条件的时候，则执行对应 case 块的代码。当没有 case 满足的时候，Select 逻辑块也会阻塞程序，当有多个 case 满足的时候，Select 会公平选择一个满足条件的 case 语句块执行。为了避免阻塞，可以使用 default 。

## 使用 sync 包

Golang 并发编程离不开 sync 包，里面提供了很多有用的类型和方法。常用知识点如下：

### Map

普通的 map 不是并发安全的，容易出现竞争问题。sync.Map 是官方提供的一个并发安全的 Map，不需要自己额外处理锁、协调等操作。该类型提供了以下方法：

- Loads(): 装载键值（获取键值）
- Store(): 储存键值
- Delete(): 删除键值
- LoadOrStore(): 如果存在则获取键值，不存在则存储
- Range(): 遍历 Map

我在项目中使用该类型替代了自己原本定义的一个全局 Map 类型，相比自己维护互斥锁或者读写锁等操作方便了不少，而且性能有优势（具体没有测试），据官方说明该 Map 类型显著减少了锁的争用。

### Mutex

互斥锁。对于加了锁的操作，其他的 Goroutine 在读取相应对象时会阻塞，知道互斥对象释放。频繁、大量的使用互斥锁会导致性能的降低，应该只在必须用的地方用。互斥锁使用不当还有可能造成死锁、活锁、饿死等情况，就比较复杂了。

### RWMutex

读写锁。相较互斥锁，读写锁相较互斥锁性能损耗低些，因为对于只读操作只加共享锁，是支持并发的，仅在写操作上加互斥锁，明显提升了效率。

### Once

Once 这个对象很有用，它是一个只执行一个操作的对象。该对象仅有一个 Do() 方法，仅当 Once 实例第一次调用 Do() 方法时，Do() 才会调用传递过来的函数。

这对于要实现单例模式的代码会方便一些，定义一个 Once 对象，将要实现单例的对象的初始化方法传递给 Do()，就能保证单例的并发安全。

但使用互斥锁不也能解决这个问题吗？其实不然，互斥锁的代价太高，会导致 Goroutines 无法对该变量进行并发访问。那我把它改成读写锁呢？例如：

```
var mu sync.RWMutex // 定义读写锁
var someMap map[string]interface{}

// 并发安全的单例
func GetMap(name string) interface{} {
    // 加读锁，并判断 someMap 是否为空
    // 如果不为空则获取 Map 中的值
    mu.RLock()
    if someMap != nil {
        foo := someMap[name]
        mu.RUnlock()
        return foo
    }
    // 注意这里无法在 else 语句将共享锁升级为互斥锁
    mu.RUnlock()

    // 如果 someMap 为空加互斥锁，并初始化 someMap
    mu.Lock()

    // 注意：必须要再一次检查是否为 nil
    if someMap == nil { 
        CreateMap()
    }
    foo := someMap[name]
    mu.Unlock()
    return foo
}
```

然而这种方式不觉得太复杂了吗？

# Pool

Pool 是一组用来保存或者检索的临时对象。Pub() 方法的入参是空接口，意味着可以把任何类型的对象放置在池子中以复用，减轻垃圾回收器的压力，正确的使用能很轻松的构建高效、线程安全的自由列表。

# WaitGroup

WaitGroup 适用于等待多个 Goroutine 完成的逻辑下使用，控制多个 Goroutine 之间的同步。每新建一个 Goroutine 就可以调用 WaitGroup.Add() 来增加需要等待的 Goroutine 数量，当每个 Goroutine 都运行完毕时各自调用 Done() ，可以在程序的外层调用 Wait() 以阻塞程序直到所有 Goroutine 都调用了 Done()。

这样就可以在程序中使每个 WaitGroup 之间是同步的，但是 WaitGroup 之内的 Goroutine 是异步的。

# Context

与 WaitGroup 不同，Context 实现了对串行的 Goroutine 的跟踪和控制，比如一个 Goroutine 在运行过程中派生出了其他 Goroutine，这些派生出的 Goroutine 又派生出了其他 Goroutine，这种 Goroutine 的关系链使用之前的 Channel + Select 来维护会使得程序变得异常复杂。而使用 Context 上下文来实现 Goroutine 间截止时间、取消信号或其他变量的共享和传递会变得简单。

Context 有四种方法来派生出不同的上下文：

- WithCancel: 传递取消信号
- WithDeadline: 传递截止时间
- WithTimeout: 传递超时信号
- WithValue: 传递其他值

例如 WithCancel 派生出的上下文，在对应 Goroutine 中调用 cancel() 会使所有同一上下文派生出的所有 Goroutine 关闭，在派生出的 Goroutine 中可以通过监听 ctx.Done() 来判断是否关闭 Goroutine。

Context 还有许多其他的用途，在 gRPC 、gin 中大量运用了 Context。Context 是并发安全的，所以可以放心的在 Goroutines 之间传递。

在Golang并发编程中，内存泄露和一些注意事项是需要特别关注的，以下是常见的情况和建议：

# 常见内存泄露及注意事项

1. **goroutine泄露**
   - **场景**：goroutine因阻塞（如未关闭的channel、死循环）无法退出，导致内存泄露。每个goroutine占用2KB内存，大量goroutine泄露会造成内存占用持续增加。
   - **原因**：goroutine在执行时被阻塞而无法退出，如未关闭的channel导致接收方一直等待。
   - **避免**：确保goroutine有明确的退出机制，如使用`context.Context`传递取消信号，关闭channel通知协程退出。

2. **time.Ticker未关闭**
   - **场景**：time.Ticker每间隔指定时间向通道写数据，若不调用`Stop()`方法，会一直占用内存。
   - **避免**：在使用完time.Ticker后，务必调用`Stop()`方法来停止它。

3. **字符串和切片截取**
   - **场景**：长字符串或切片被截取后，如果截取的小部分还在活跃，原大块内存将无法被回收，导致临时内存泄露。
   - **避免**：在不需要保留原大块内存时，可以复制需要的部分到新的变量，或者使用其他方式避免共享底层数组。

4. **Finalizer导致泄漏**
   - **场景**：设置Finalizer后，对象的内存无法被垃圾回收器及时回收，尤其当存在循环引用时。
   - **避免**：谨慎使用Finalizer，避免不必要的循环引用。

5. **Deferring Function Call导致泄漏**
   - **场景**：大量文件打开后仅在函数结束时释放，造成临时内存泄露。
   - **避免**：在循环中打开文件时，应立即关闭，而不是等待函数结束。

6. **内存分配未释放**
   - **场景**：持续分配内存但未释放，如持续向切片中添加数据而未清理。
   - **避免**：合理管理内存分配和释放，避免不必要的内存堆积。

7. **大数组作为参数**
   - **场景**：大数组作为形参时，值拷贝导致内存使用激增。
   - **避免**：对于大数组，使用切片或指针传递，减少内存占用。

## 注意事项

1. **并发安全与同步**
   - 共享资源需用`sync.Mutex`、`sync.RWMutex`或原子操作保护，优先使用Channel传递数据，避免直接共享内存。

2. **控制并发量**
   - 避免无限制创建协程，使用带缓冲的Channel、Worker Pool或信号量限制并发数。

3. **处理协程错误**
   - 在goroutine内部使用`defer`+`recover`处理`panic`，避免整个程序崩溃。

4. **避免Channel死锁**
   - 确保Channel的发送和接收成对出现，避免协程永久阻塞，如未关闭的Channel可能导致接收方等待。

5. **利用Context传递元数据和取消信号**
   - 使用`context.Context`传递元数据和取消信号，方便控制goroutine的生命周期。

6. **合理使用锁**
   - 避免在持有锁的情况下调用可能导致阻塞的操作，如I/O操作，防止死锁。

7. **监控和分析**
   - 使用`runtime.MemStats`监控内存使用情况，使用`pprof`进行性能分析，及时发现和解决内存和性能问题。

通过遵循以上建议和注意事项，可以有效避免Golang并发编程中的内存泄露问题，编写出高效、稳定的并发程序。