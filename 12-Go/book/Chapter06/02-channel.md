
Channel
=========

- channel 为引用类型（同 map，slice。注意 array 为值类型），使用 make 关键字创建。channel 是个 FIFO 队列，用于多个 goroutine 间通信。其内部实现了同步，可确保并发安全。

### 不带缓冲的 channel
- channel 默认为同步模式（即不带缓冲的 channel）。同步操作需要发送和接收配对，否则会被阻塞，直到另一方准备好之后被唤醒，即调用写操作之后被阻塞住（不管 channel 是否为空），直到该数据被读取掉。调用读操作后，如果 channel 中有数据则被读出，如果为空则阻塞住，直到向 EnQueue 中存数据。

```go
func main() {
    data := make(chan int)          // 数据交换队列
    exit := make(chan bool)         // 退出通知

    go func() {
        for d := range data {       // 从队列迭代接收数据，直到 close
            fmt.Println(d)
        }

        fmt.Println("recv over.")
        exit <- true                // 发出退出通知
    }()

    data <- 1                       // 发送数据
    data <- 2
    data <- 3
    close(data)                     // 关闭队列

    fmt.Println("send over.")
    <-exit                          // 等待退出通知
    fmt.Println("exit now.")
}

// 输出：
1
2
3
send over.
recv over.
exit now.
```

### 带缓冲的 Channel
- 异步方式通过判断缓冲区来决定是否租塞。如果缓冲区已满，发送被阻塞；缓冲区为空，接收被阻塞。

- 通常情况下，异步 channel 可减少排队阻塞，具备更高的效率。但应考虑使用指针规避大对象拷贝，将多个元素打包，减少缓冲区大小等。

```go
func main() {
    data := make(chan int, 3)   // 缓冲区可存储3个元素
    exit := make(chan bool)

    data <- 1                   // 在缓冲区未满前，不会阻塞
    data <- 2
    data <- 3

    go func() {
        for d := range data {   // 在缓冲区未空前，不会阻塞
            fmt.Println(d)
        }

        exit <- true
    }()

    data <- 4                   // 如果缓冲区已满，阻塞
    data <- 5
    close(data)

    <-exit
}
```

- 除 range 外，还可用 ok-idiom 模式判断 channel 是否关闭。

```go
for {
    if d, ok := <- data; ok {
        fmt.Println(d)
    } else {
        break
    }
}
```

- 向 closed channel 发送数据引发 panic 错误，接收立即返回零值。而 nil channel，无论收发都会被租塞。

- 内置函数 len 返回未被读取的缓冲元素数量，cap 返回缓冲区大小。

```go
d1 := make(chan int)
d2 := make(chan int, 3)

d2 <- 1

fmt.Println(len(d1), cap(d1))    // 0  0
fmt.Println(len(d2), cap(d2))    // 1  3
```

### 单向 channel
- 可将 channel 隐式转换为单向队列，只收或只发。

```go
c := make(chan int, 3)

var send chan<- int = c     // send-only
var recv <-chan int = c     // receive-only

send <- 1
// <-send                   // Error: receive from send-only type chan<- int

<-recv
// recv <- 2                // Error: send to receive-only type <-chan int
```

- 单向 channel 无法转换为普通 channel。

```go
d := (chan int)(send)       // Error: cannot convert type chan<- int to type chan int
d := (chan int)(recv)       // Error: cannot convert type <-chan int to type chan int
```

### select
- select 关键字仅用于 channel 中，常用来处理多个 channel。select 会尝试执行各个 case，如果都可以执行则随机选一个执行，否则执行 default case。

```go
func main() {
    a, b := make(chan int, 3), make(chan int)
    go func() {
        v, ok, s := 0, false, ""
        for {
            select {
            case v, ok = <-a: s = "a"
            case v, ok = <-b: s = "b"
            }

            if ok {
                fmt.Println(s, v)
            } else {
                os.Exit(0)
            }
        }
    }()

    for i := 0; i < 5; i++ {
        select {                      // 随机选择可用 channel，发送数据
        case a <- i:
        case b <- i:
        }
    }

    close(a)
    select {}                        // 无可用 channel，阻塞 main goroutine
}

// 输出：
b 3
a 0
a 1
a 2
b 4
```

- channel 也可作为结构成员。

```go
type Request struct {
    data []int
    ret chan int
}

func NewRequest(data ...int) *Request {
    return &Request{ data, make(chan int, 1) }
}

func Process(req *Request) {
    x := 0
    for _, i := range req.data {
        x += i
    }
    req.ret <- x
}

func main() {
    req := NewRequest(10, 20, 30)
    Process(req)
    fmt.Println(<-req.ret)
}
```

### 常用应用场景

- 用 channel 实现信号量 semaphore。

```go
func main() {
    wg := sync.WaitGroup{}
    wg.Add(3)

    sem := make(chan int, 1)

    for i := 0; i < 3; i++ {
        go func(id int) {
            defer wg.Done()

            sem <- 1

            for x := 0; x < 3; x++ {
                fmt.Println(id, x)
            }

            <-sem
        }(i)
    }

    wg.Wait()
}

// 输出：
$ GOMAXPROCS=2 go run main.go
0 0
0 1
0 2
1 0
1 1
1 2
2 0
2 1
2 2
```

- 用 closed channel 发出退出通知。

```go
func main() {
    var wg sync.WaitGroup
    quit := make(chan bool)

    for i := 0; i < 2; i++ {
        wg.Add(1)

        go func(id int) {
        defer wg.Done()

        task := func() {
            println(id, time.Now().Nanosecond())
            time.Sleep(time.Second)
        }

        for {
            select {
            case <-quit:           // closed channel 不会阻塞，可用作退出通知
                return
            default:               // 执行正常任务
                task()
            }
        }
        }(i)
    }

    time.Sleep(time.Second * 5)    // 让测试 goroutine 运行一段时间
    close(quit)                    // 发出退出通知
    wg.Wait()
}
```

- 用 select 实现超时。

```go
func main() {
    w := make(chan bool)
    c := make(chan int, 2)
    go func() {
        select {
        case v := <-c: fmt.Println(v)
        case <-time.After(time.Second * 3): fmt.Println("timeout.")
        }

        w <- true
    }()

    // c <- 1        // 注释掉，引发 timeout
    <-w
}
```

- 生成器（类似于 lazy load）。

```go
func xrange() chan int{                 // xrange 用来生成自增的整数
    var ch chan int = make(chan int)

    go func() {                         // 开启一个 goroutine
        for i := 0; ; i++ {
            ch <- i                     // 直到信道索要数据，才把i添加进信道
        }
    }()

    return ch
}

func main() {

    generator := xrange()

    for i:=0; i < 1000; i++ {           // 生成 1000 个自增的整数
        fmt.Println(<-generator)
    }
}
```

- 服务化

```go
/*
 * 一般业务中消息数据来自一个独立的服务，这个服务只负责 返回某个用户的新的消息提醒。
 */
func get_notification(user string) chan string{

    notifications := make(chan string)
    // 此处可以查询数据库获取新消息等
    go func() { // 悬挂一个信道出去
        notifications <- fmt.Sprintf("Hi %s, welcome to weibo.com!", user)
    }()

    return notifications
}

func main() {
    jack := get_notification("jack") // 获取 jack 的消息
    joe := get_notification("joe")   // 获取 joe 的消息

    // 获取消息的返回
    fmt.Println(<-jack)
    fmt.Println(<-joe)
}
```

- 多路复合。可以把多个信道的数据合并到一个信道。但需要按顺序输出返回值（先进先出）

```go
/*
 * 假设要计算很复杂的一个运算 100-x ,分为三路计算，最后统一在一个信道中取出结果:
 */
func do_stuff(x int) int {                     // 一个比较耗时的事情，比如计算
    time.Sleep(time.Duration(rand.Intn(10)) * time.Millisecond) //模拟计算
    return 100 - x                             // 假如100-x是一个很费时的计算
}

 func branch(x int) chan int{                   // 每个分支开出一个 goroutine做计算并把计算结果流入各自信道
    ch := make(chan int)
    go func() {
        ch <- do_stuff(x)
    }()
    return ch
}

func fanIn(chs... chan int) chan int {
    ch := make(chan int)

    for _, c := range chs {
        // 注意此处明确传值
        go func(c chan int) {ch <- <- c}(c)  // 复合
    }

    return ch
}

// 上述函数也可用 select, 这样只需要一个 goroutine 来取数据:
// func fanIn(branches ... chan int) chan int {
//     c := make(chan int)

//     go func() {
//         for i := 0 ; i < len(branches); i++ {
//             select {                            // select 会尝试着依次取出各个信道的数据
//             case v1 := <- branches[i]: c <- v1
//             }
//         }
//     }()

//     return c
// }

func main() {
    result := fanIn(branch(1), branch(2), branch(3))

    for i := 0; i < 3; i++ {
        fmt.Println(<-result)
    }
}
```

- Daisy-chain
```go
/*
 *  利用信道菊花链筛法求某一个整数范围的素数
 *  筛法求素数的基本思想是：把从 1 开始的、某一范围内的正整数从小到大顺序排列，
 *  1不是素数，首先把它筛掉。剩下的数中选择最小的数是素数，然后去掉它的倍数。
 *  依次类推，直到筛子为空时结束
 */
package main

import "fmt"

func xrange() chan int{ // 从 2 开始自增的整数生成器
    var ch chan int = make(chan int)

    go func() { // 开出一个goroutine
        for i := 2; ; i++ {
            ch <- i  // 直到信道索要数据，才把i添加进信道
        }
    }()

    return ch
}


func filter(in chan int, number int) chan int {
    // 输入一个整数队列，筛出是 number 倍数的, 不是 number 的倍数的放入输出队列
    // in: 输入队列
    out := make(chan int)

    go func() {
        for {
            i := <- in   // 从输入中取一个

            if i % number != 0 {
                out <- i // 放入输出信道
            }
        }
    }()

    return out
}


func main() {
    const max = 100   // 找出 100 以内的所有素数
    nums := xrange()  // 初始化一个整数生成器
    number := <-nums  // 从生成器中抓一个整数(2), 作为初始化整数

    for number <= max {              // number 作为筛子，当筛子超过 max 时结束筛选
        fmt.Println(number)          // 打印素数, 筛子即一个素数
        nums = filter(nums, number)  //筛掉number的倍数
        number = <- nums             // 更新筛子
    }
}
```

- 定时器

```go
/*
 * 利用信道做定时器
 */

package main

import (
    "fmt"
    "time"
)


func timer(duration time.Duration) chan bool {
    ch := make(chan bool)


    go func() {
        time.Sleep(duration)
        ch <- true
    }()

    // time.Sleep(duration)
    // ch <- true
    // Error: deadLock 到这里已经阻塞，不会执行下一步
    return ch
}

func main() {
    timeout := timer(time.Second)      // 定时 1s

    for {
        select {
        case <- timeout:
            fmt.Println("already 1s!") // 到时间
            return                     // 结束程序
        }
    }
}
```

### 协程泄露

协程和内存一样，是系统资源。对于内存，有自动垃圾回收。但是对于协程，没有相应的回收机制。
一般而言，协程执行结束后就会销毁。协程也会占用内存，如果发生协程泄漏，影响和内存泄漏一样严重。轻则拖慢程序，重则压垮机器。

有两种情况会导致协程无法结束：

- 协程想从一个通道读数据，但没有向这个通道写入数据。

解决方法：加入超时机制。对于有不确定会不会返回的情况，必须加入超时，避免出现永久等待。另外不一定要使用定时器才能终止协程。也可以对外暴露一个退出提醒通道。任何其他协程都可以通过该通道来提醒这个协程终止。

- 协程想往一个通道写数据，但没有监听这个通道，该协程将永远无法向下执行。

解决方法：给通道加缓冲。但前提是这个通道只会接收到固定数目的写入。如已知一个通道最多只会接收 N 次数据，那么就将这个通道的缓冲设置为 N。
那么该通道将永远不会堵塞，协程自然也不会泄漏。也可以将其缓冲设置为无限，但这样就要承担内存泄漏的风险。等协程执行完毕后，这部分通道内存将会失去引用，会被自动回收。
