下面给一个 Windows API Event 的完整示例：

场景：

- 主线程（main）负责初始化任务。
- 两个 worker 线程启动后等待 Event。
- 主线程调用 `SetEvent()` 通知两个 worker 开始工作。
- 使用 **Manual-reset Event**，因为希望一次通知两个 worker。

### 示例：Manual-reset Event 广播启动两个线程

```cpp
#include <windows.h>
#include <iostream>
#include <thread>

HANDLE g_startEvent = nullptr;

void worker(int id)
{
    std::cout << "Worker " << id << " started, waiting for event...\n";

    // 等待主线程发送启动信号
    DWORD ret = WaitForSingleObject(
        g_startEvent,
        INFINITE
    );

    if (ret == WAIT_OBJECT_0)
    {
        std::cout << "Worker " << id 
                  << " received event, working...\n";

        // 模拟工作
        Sleep(1000);

        std::cout << "Worker " << id 
                  << " finished.\n";
    }
    else
    {
        std::cout << "Worker " << id 
                  << " wait failed.\n";
    }
}


int main()
{
    /*
       创建 Manual-reset Event

       参数:
       bManualReset = TRUE
          SetEvent 后保持 signaled 状态

       bInitialState = FALSE
          初始为 non-signaled
    */
    g_startEvent = CreateEvent(
        nullptr,
        TRUE,
        FALSE,
        nullptr
    );

    if (!g_startEvent)
    {
        std::cerr << "CreateEvent failed\n";
        return 1;
    }


    // 创建两个 worker
    std::thread worker1(worker, 1);
    std::thread worker2(worker, 2);


    // 等待一下，让 worker 进入 WaitForSingleObject
    Sleep(1000);


    std::cout << "Main thread: SetEvent()\n";

    // 广播通知所有等待线程
    SetEvent(g_startEvent);


    worker1.join();
    worker2.join();


    CloseHandle(g_startEvent);

    std::cout << "Main thread exit\n";

    return 0;
}
```

可能输出：

```
Worker 1 started, waiting for event...
Worker 2 started, waiting for event...
Main thread: SetEvent()
Worker 1 received event, working...
Worker 2 received event, working...
Worker 1 finished.
Worker 2 finished.
Main thread exit
```

------

### 如果改成 Auto-reset Event

把：

```cpp
g_startEvent = CreateEvent(
    nullptr,
    TRUE,
    FALSE,
    nullptr
);
```

改成：

```cpp
g_startEvent = CreateEvent(
    nullptr,
    FALSE,   // auto-reset
    FALSE,
    nullptr
);
```

那么：

```cpp
SetEvent(g_startEvent);
```

只会唤醒 **一个 worker**：

```
Main thread: SetEvent()

Worker 1 received event, working...
Worker 2 still waiting...
```

因为 Auto-reset 的行为：

```
       SetEvent()
            |
            v
    Event = signaled
            |
            v
    唤醒一个等待线程
            |
            v
    自动 Reset
            |
            v
    Event = non-signaled
```

------

### 这个例子对应实际应用中的什么场景？

Manual-reset Event：

```
主线程
 |
 | 初始化完成
 |
 SetEvent()
 |
 +----------+
 |          |
 v          v
worker1   worker2
开始工作  开始工作
```

典型用途：

- 服务启动通知
- 多线程程序初始化屏障
- 配置加载完成通知
- 网络服务 ready 信号

Auto-reset Event：

```
producer

   SetEvent()
        |
        v

   一个 worker 消费任务
```

典型用途：

- 单任务唤醒
- 工作线程池通知
- 简单 producer-consumer 模型

不过现代 Windows 高性能线程池通常会使用更专业的机制，例如：

- IOCP（I/O Completion Port）
- Condition Variable
- SRW Lock
- WaitOnAddress

Event 更多用于**跨线程生命周期通知、状态变化通知和简单同步控制**。