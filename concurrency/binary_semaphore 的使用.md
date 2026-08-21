C++ std::binary_semaphore 改为 std::condition_variable + std::mutex + state变量



```c

//#include <print>
//#include <semaphore>
//#include <thread>
//
//int main() {
//    std::binary_semaphore startSignal{ 0 };
//    std::binary_semaphore doneSignal{ 0 };
//
//    int result = 0;
//
//    std::jthread worker([&] {
//        startSignal.acquire();
//
//        result = 42;
//
//        doneSignal.release();
//        });
//
//    std::println("[main] starting worker"); 
//    startSignal.release();
//
//    doneSignal.acquire();
//    std::println("[main] result = {}", result);
//}


#include <condition_variable>
#include <mutex>
#include <print>
#include <thread>

int main()
{
    std::mutex mtx;
    std::condition_variable cv;

    bool started = false;
    bool done = false;

    int result = 0;

    std::jthread worker([&] {
        // 等待 main 发出 start 信号
        {
            std::unique_lock lock(mtx);

            cv.wait(lock, [&] {
                return started;
                });
        }

        // worker 执行任务
        result = 42;

        // 通知 main：worker 已经完成
        {
            std::lock_guard lock(mtx);
            done = true;
        }

        cv.notify_one();
        });

    
    
    
    std::println("[main] starting worker");

    // 通知 worker 开始
    {
        std::lock_guard lock(mtx);
        started = true;
    }

    cv.notify_one();

    // 等待 worker 完成
    {
        std::unique_lock lock(mtx);

        cv.wait(lock, [&] {
            return done;
            });
    }

    std::println("[main] result = {}", result);
}
```



注意：

1. condition_variable 不保存状态；
2. 注意虚假唤醒；

### 为什么 `cv.wait()` 要有 `started`？

这是 `condition_variable` 和 semaphore 最核心的区别之一。

不能这样写：

```
cv.wait(lock);
```

然后认为：

```
cv.notify_one();
```

就是一个可靠的“信号”。

因为 `condition_variable` 允许 **spurious wakeup（虚假唤醒）**。

例如：

```
worker                         main

cv.wait()
   │
   │                         notify_one()
   │
   ├──────── 被唤醒 ──────────┤
   │
   ↓
继续执行
```

但也可能：

```
worker

cv.wait()
   │
   │
   ├── 虚假唤醒
   │
   ↓
```

所以必须检查一个真正的条件：

```
cv.wait(lock, [] {
    return started;
});
```

其实际逻辑相当于：

```
while (!started) {
    cv.wait(lock);
}
```

因此这里的：

```
bool started = false;
```

实际上就是 condition variable 的“状态”。