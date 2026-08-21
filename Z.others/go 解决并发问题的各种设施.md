除了 `select`，Go 标准库和生态里还有不少配合 goroutine + channel 使用的设施，各自解决不同的并发问题。系统梳理一下：

## 1. `sync.WaitGroup` —— 等待一组 goroutine 全部完成

最常用的场景：启动 N 个 goroutine，等它们全部跑完再继续。

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Println("worker", id, "done")
    }(i)
}

wg.Wait() // 阻塞直到所有 goroutine 调用了 Done
fmt.Println("all workers finished")
```

跟 channel 配合：常用来"等收集完所有结果再关闭 channel"：

```go
results := make(chan int, 10)
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        results <- n * n
    }(i)
}

go func() {
    wg.Wait()
    close(results) // 所有 goroutine 写完后关闭，接收端才能用 for-range 正常退出
}()

for r := range results {
    fmt.Println(r)
}
```

## 2. `sync.Mutex` / `sync.RWMutex` —— 保护共享内存

如果多个 goroutine 要访问同一个变量（而不是通过 channel 通信），用锁：

```go
var mu sync.Mutex
counter := 0

var wg sync.WaitGroup
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        mu.Lock()
        counter++
        mu.Unlock()
    }()
}
wg.Wait()
fmt.Println(counter) // 100
```

Go 的哲学是"不要通过共享内存来通信，而要通过通信来共享内存"（channel 优先），但涉及高频读写共享状态时，锁往往比 channel 更简单高效。`RWMutex` 适合读多写少场景（`RLock`/`RUnlock` 允许多个读者并发）。

## 3. `sync/atomic` —— 无锁的原子操作

比 Mutex 更轻量，适合简单的计数器、标志位：

```go
var counter int64

var wg sync.WaitGroup
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        atomic.AddInt64(&counter, 1)
    }()
}
wg.Wait()
fmt.Println(atomic.LoadInt64(&counter))
```

Go 1.19+ 还提供了类型化的原子类型，更安全：

```go
var counter atomic.Int64
counter.Add(1)
fmt.Println(counter.Load())
```

## 4. `sync.Once` —— 确保某段代码只执行一次

常用于并发安全的单例初始化：

```go
var once sync.Once
var config *Config

func getConfig() *Config {
    once.Do(func() {
        config = loadConfig() // 无论多少 goroutine 同时调用，只会执行一次
    })
    return config
}
```

## 5. `context.Context` —— 取消信号 + 超时控制（跟 channel 深度绑定）

这是**和 channel/select 结合最紧密**的设施，本质上 `ctx.Done()` 返回的就是一个 channel：

```go
func worker(ctx context.Context, ch chan int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("worker 收到取消信号:", ctx.Err())
            return
        case v := <-ch:
            fmt.Println("处理:", v)
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    ch := make(chan int)
    go worker(ctx, ch)

    ch <- 1
    ch <- 2
    time.Sleep(5 * time.Second) // 超过 3 秒后，worker 会自动退出
}
```

常见的 `context` 用法：

- `context.WithCancel(parent)` —— 手动调用 `cancel()` 触发取消
- `context.WithTimeout(parent, d)` —— 超时自动取消
- `context.WithDeadline(parent, t)` —— 指定截止时间
- `context.WithValue(parent, key, val)` —— 传递请求范围的数据（不建议滥用）

这是**取消一组相关 goroutine 的标准做法**——尤其是在 HTTP 请求、RPC 调用链路里非常重要。

## 6. 缓冲 channel 当信号量（Semaphore）—— 限制并发数

```go
sem := make(chan struct{}, 3) // 最多允许 3 个并发

var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        sem <- struct{}{}        // 获取一个"名额"，满了就阻塞
        defer func() { <-sem }() // 释放名额

        fmt.Println("处理任务", id)
        time.Sleep(time.Second)
    }(i)
}
wg.Wait()
```

这是限制"最多同时跑 N 个 goroutine"（比如控制并发请求数、数据库连接数）的经典写法。

## 7. `golang.org/x/sync/errgroup` —— WaitGroup 的加强版，支持错误传播和取消

比原生 `WaitGroup` 好用很多，任何一个 goroutine 出错都能让整组提前取消：

```go
import "golang.org/x/sync/errgroup"

func main() {
    g, ctx := errgroup.WithContext(context.Background())

    urls := []string{"url1", "url2", "url3"}
    for _, url := range urls {
        url := url
        g.Go(func() error {
            select {
            case <-ctx.Done():
                return ctx.Err()
            default:
            }
            return fetch(url) // 返回 error
        })
    }

    if err := g.Wait(); err != nil {
        fmt.Println("有任务失败:", err)
    }
}
```

这是实际项目里替代手写 `WaitGroup + error channel` 的首选方案。

## 8. `golang.org/x/sync/semaphore` —— 更规范的信号量（比 buffered channel 语义更清晰）

```go
import "golang.org/x/sync/semaphore"

sem := semaphore.NewWeighted(3) // 最多3个并发

for i := 0; i < 10; i++ {
    sem.Acquire(context.Background(), 1)
    go func(id int) {
        defer sem.Release(1)
        fmt.Println("处理", id)
    }(i)
}
```

支持带权重的资源控制（比如某些任务占用2个名额），也支持 `context` 取消等待。

## 9. `golang.org/x/sync/singleflight` —— 合并重复请求

多个 goroutine 同时请求同一个 key 时，只真正执行一次，其余等待共享结果（常用于缓存击穿保护）：

```go
import "golang.org/x/sync/singleflight"

var g singleflight.Group

func getUser(id string) (interface{}, error) {
    v, err, _ := g.Do(id, func() (interface{}, error) {
        return queryDB(id) // 相同 id 并发调用只会真正执行一次
    })
    return v, err
}
```

## 10. `time.Timer` / `time.Ticker` —— 定时器（前面例子已用到）

- `time.After(d)` —— 一次性，返回 channel，超时后发送当前时间
- `time.NewTimer(d)` —— 同上，但可以 `Stop()`/`Reset()`
- `time.NewTicker(d)` —— 周期性触发

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

for i := 0; i < 3; i++ {
    <-ticker.C
    fmt.Println("tick")
}
```

## 11. Worker Pool 模式（组合 channel + WaitGroup，非新设施但很常用）

```go
func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for j := range jobs {
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    var wg sync.WaitGroup

    for w := 1; w <= 3; w++ { // 3 个 worker
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    for j := 1; j <= 9; j++ {
        jobs <- j
    }
    close(jobs)

    wg.Wait()
    close(results)

    for r := range results {
        fmt.Println(r)
    }
}
```

## 总结：按场景选工具

| 需求                        | 用什么                                       |
| --------------------------- | -------------------------------------------- |
| 等待一组 goroutine 全部结束 | `sync.WaitGroup`                             |
| 保护共享变量的并发读写      | `sync.Mutex` / `sync.RWMutex`                |
| 简单计数器、标志位          | `sync/atomic`                                |
| 只初始化一次                | `sync.Once`                                  |
| 取消信号、超时控制          | `context.Context`                            |
| 限制并发数量                | 带缓冲 channel 当信号量 / `x/sync/semaphore` |
| 一组任务，出错就全部取消    | `x/sync/errgroup`                            |
| 合并重复的并发请求          | `x/sync/singleflight`                        |
| 定时/周期性任务             | `time.Timer` / `time.Ticker`                 |
| 多路等待 channel 事件       | `select`（前面已详细讲过）                   |

实际工程里最常见的组合是：**`context` 控制取消/超时 + `errgroup` 管理一组 goroutine + channel 传递数据**，这三者搭配几乎能覆盖大部分并发场景。