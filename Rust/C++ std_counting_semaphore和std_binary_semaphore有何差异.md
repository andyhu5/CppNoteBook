### C++ std::counting_semaphore和std::binary_semaphore有何差异



在 C++20 中，std::counting_semaphore 和 std::binary_semaphore 都是标准库提供的信号量（semaphore）类，用于线程同步，但它们有以下关键差异：1. 计数范围

- std::counting_semaphore:
  - 是一个通用信号量，允许计数器取值范围为 [0, max()]，其中 max() 是模板参数指定的最大值。
  - 可以表示多个资源单位（计数），适用于需要控制对多个资源的访问场景。
  - 例如，std::counting_semaphore<10> 表示最多允许 10 个线程同时访问某个资源。
  - 计数器可以动态增加（通过 release(n)）或减少（通过 acquire() 或 try_acquire()）。
- std::binary_semaphore:
  - 是 std::counting_semaphore<1> 的特化版本，计数器只能取值 0 或 1。
  - 专门用于二元场景（类似于互斥锁），表示资源要么可用（计数为 1），要么不可用（计数为 0）。
  - 功能上等价于 std::counting_semaphore<1>，但语义更明确，表示专为二值信号量设计。
- 用途和语义

- std::counting_semaphore:
  - 适用于需要控制多个资源或允许多个线程并发访问的场景，例如线程池、资源池（如数据库连接池）或限制并发访问数量。
  - 语义更通用，表示资源计数，适合复杂同步需求。
  - 示例：限制最多 5 个线程同时访问某个资源。
- std::binary_semaphore:
  - 适用于简单的互斥或信号通知场景，例如生产者-消费者模型中的单次信号传递，或实现简单的锁机制。
  - 语义明确，强调二值状态（类似于条件变量或简单的锁）。
  - 示例：一个线程通知另一个线程某个事件已发生。
- 接口差异

- 两者接口基本相同，都支持 acquire()、try_acquire()、release(n) 等操作。
- 但 std::binary_semaphore 的计数器被限制为 0 或 1，因此 release(n) 中 n 通常为 1（否则可能抛出异常或未定义行为，具体取决于实现）。
- std::counting_semaphore 允许 release(n) 增加任意计数（不超过 max()），更灵活。
- 性能与实现

- std::counting_semaphore:
  - 实现上需要管理更大的计数范围，可能涉及更复杂的内部逻辑（如原子操作或锁）。
  - 适用于需要动态调整资源计数的场景，但可能略微增加开销。
- std::binary_semaphore:
  - 由于计数器只有 0 和 1，实现可以更简单，优化空间更大（例如，可能直接映射到某些平台的原子操作或轻量级锁）。
  - 在某些场景下可能比 std::counting_semaphore<1> 更高效，但实际性能差异取决于编译器和平台。
- 代码示例std::counting_semaphore

cpp



```cpp
#include <iostream>
#include <thread>
#include <semaphore>

std::counting_semaphore<3> sem(3); // 最多允许 3 个线程同时访问

void worker(int id) {
    sem.acquire();
    std::cout << "Thread " << id << " acquired resource\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Thread " << id << " releasing resource\n";
    sem.release();
}

int main() {
    std::thread t1(worker, 1), t2(worker, 2), t3(worker, 3), t4(worker, 4);
    t1.join(); t2.join(); t3.join(); t4.join();
}
```

输出示例：





```text
Thread 1 acquired resource
Thread 2 acquired resource
Thread 3 acquired resource
Thread 1 releasing resource
Thread 4 acquired resource
Thread 2 releasing resource
Thread 3 releasing resource
Thread 4 releasing resource
```

- 最多 3 个线程同时获取资源，第 4 个线程需等待。

std::binary_semaphore

cpp



```cpp
#include <iostream>
#include <thread>
#include <semaphore>

std::binary_semaphore sem(1); // 初始值为 1

void worker(int id) {
    sem.acquire();
    std::cout << "Thread " << id << " acquired semaphore\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Thread " << id << " releasing semaphore\n";
    sem.release();
}

int main() {
    std::thread t1(worker, 1), t2(worker, 2);
    t1.join(); t2.join();
}
```

输出示例：





```text
Thread 1 acquired semaphore
Thread 1 releasing semaphore
Thread 2 acquired semaphore
Thread 2 releasing semaphore
```

- 每次只有一个线程能获取信号量，类似互斥锁。
- 选择哪一个？

- 如果你需要控制对多个资源的访问或允许多个线程并发操作，使用 std::counting_semaphore。
- 如果你只需要简单的二值同步（如互斥或单次信号通知），使用 std::binary_semaphore，因为它语义更清晰，可能更高效。
- 如果只需要 0 或 1 的信号量，std::binary_semaphore 是更好的选择，因为它明确表达了二值语义，代码更易读。

总结

- std::counting_semaphore：通用信号量，支持多资源计数，适合复杂资源管理。
- std::binary_semaphore：特化为二值信号量，适合简单互斥或信号通知，语义更明确，可能更高效。
- 两者功能上 std::binary_semaphore 等价于 std::counting_semaphore<1>，但选择时应根据场景和语义清晰度来决定。
