这是个经典需求——"事件驱动"的 goroutine：平时挂起不占 CPU，等外部信号一来才被唤醒执行。Go 里有几种标准做法，按场景选择。

## 方案一：无缓冲 channel 作为"信号"（最常用、最简单）

适合：只触发一次，触发后 goroutine 执行完就结束。

```go
func main() {
	trigger := make(chan struct{}) // struct{} 是零内存开销的"信号"类型

	go func() {
		fmt.Println("goroutine 已启动，进入休眠，等待事件...")
		<-trigger // 阻塞在这里，不占用 CPU，直到有人往 trigger 发送信号
		fmt.Println("事件触发！开始执行任务")
		// ... 真正的工作逻辑
	}()

	time.Sleep(3 * time.Second) // 模拟主线程过一会儿才触发事件
	fmt.Println("主线程触发事件")
	trigger <- struct{}{} // 发送信号，唤醒 goroutine

	time.Sleep(1 * time.Second) // 等 goroutine 执行完（实际项目用 WaitGroup 更规范）
}
```

**为什么用 `chan struct{}` 而不是 `chan bool` 或 `chan int`？** 因为这里只关心"有没有信号"，不关心信号携带什么值，`struct{}` 是空结构体，不占用内存，是 Go 里表达"纯信号"的惯用类型。

## 方案二：`close(channel)` 实现"广播"式触发（可以唤醒多个 goroutine）

适合：一个事件需要同时唤醒**多个** goroutine（比如系统"就绪"信号）。

```go
func main() {
	ready := make(chan struct{})

	var wg sync.WaitGroup
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			fmt.Println("worker", id, "休眠，等待就绪信号...")
			<-ready // 阻塞等待
			fmt.Println("worker", id, "被唤醒，开始工作")
		}(i)
	}

	time.Sleep(2 * time.Second)
	fmt.Println("触发就绪信号")
	close(ready) // 关闭 channel 会让所有正在等待它的 goroutine 同时被唤醒！

	wg.Wait()
}
```

**关键原理**：对一个已关闭的 channel 执行接收操作 `<-ch`，会立即返回（不阻塞），拿到该类型的零值。所以 `close(ready)` 是一个**一次性广播机制**——所有阻塞在 `<-ready` 的 goroutine 会同时被唤醒。这跟"发送一个值"（只有一个 goroutine 能收到）不同，`close` 是"所有等待者都收到"。

## 方案三：可以重复触发多次的场景 —— 用有缓冲 channel 或循环 select

如果事件会反复触发（比如定时任务、外部通知多次），用循环 + channel：

```go
func main() {
	events := make(chan string) // 每次触发发送一次事件

	go func() {
		for {
			fmt.Println("goroutine 休眠中，等待下一个事件...")
			event := <-events // 阻塞，直到收到事件
			fmt.Println("收到事件:", event, "，开始处理")
			// 处理完，循环回到顶部，继续休眠等待下一个
		}
	}()

	time.Sleep(1 * time.Second)
	events <- "event-A"
	time.Sleep(1 * time.Second)
	events <- "event-B"
	time.Sleep(1 * time.Second)
}
```

这就是典型的"事件循环"模式——`for { <-events }`，每次处理完自动回到阻塞状态休眠，直到下一次事件到来。

## 方案四：`sync.Cond` —— 更接近传统"条件变量"语义

如果你熟悉 C/C++/Java 里的"条件变量"（`pthread_cond_wait` / `wait/notify`），Go 也有对应设施 `sync.Cond`，适合"需要检查某个共享条件是否满足"的场景：

```go
func main() {
	var mu sync.Mutex
	cond := sync.NewCond(&mu)
	ready := false

	go func() {
		mu.Lock()
		for !ready { // 用 for 而不是 if，防止"虚假唤醒"
			fmt.Println("goroutine 休眠，等待条件满足...")
			cond.Wait() // 释放锁并休眠，被唤醒后重新加锁
		}
		mu.Unlock()
		fmt.Println("条件满足，开始执行")
	}()

	time.Sleep(2 * time.Second)
	mu.Lock()
	ready = true
	mu.Unlock()
	cond.Signal() // 唤醒一个等待者；Broadcast() 唤醒所有等待者

	time.Sleep(1 * time.Second)
}
```

`cond.Wait()` 内部会自动释放锁、挂起 goroutine，被 `Signal()`/`Broadcast()` 唤醒后自动重新加锁——这跟 channel 比起来更贴近"共享状态+条件判断"的场景，但在现代 Go 代码里**用得相对少**，因为大部分场景 channel 已经够用且更符合 Go 的惯用风格。

## 方案五：`context.Context` 的 `Done()` 用作"取消/停止"类事件

如果你的"事件"具体是"停止信号"（而不是"开始信号"），更地道的做法是用 `context`：

```go
func worker(ctx context.Context) {
	fmt.Println("worker 休眠，等待取消信号...")
	<-ctx.Done() // 阻塞直到 context 被取消
	fmt.Println("收到取消信号，退出:", ctx.Err())
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go worker(ctx)

	time.Sleep(2 * time.Second)
	cancel() // 触发 ctx.Done()，唤醒 worker

	time.Sleep(1 * time.Second)
}
```

## 怎么选

| 场景                                 | 推荐方案                          |
| ------------------------------------ | --------------------------------- |
| 单次事件，唤醒一个 goroutine         | 无缓冲 channel（方案一）          |
| 单次事件，需要同时唤醒多个 goroutine | `close(channel)`（方案二）        |
| 事件会反复触发，处理完继续等下一次   | `for { <-ch }` 事件循环（方案三） |
| 需要基于共享状态判断复杂条件         | `sync.Cond`（方案四）             |
| 语义是"取消/停止"而非"开始"          | `context.Context`（方案五）       |

**核心原理都一样**：利用 channel（或 `sync.Cond`）的**阻塞接收**特性——goroutine 在没有数据/未关闭/条件不满足时，会被 Go runtime 挂起、完全不占用 CPU，等信号到来时由调度器主动唤醒，这就是"休眠直到事件激发"的本质实现方式，而不需要任何轮询。