

#### std::counting_semaphore 的 try_acquire()   try_acquire_for()  try_acquire_until()





                         是否等待
                            │
              ┌─────────────┼──────────────┐
              │             │              │
        acquire()      try_acquire()   try_acquire_for()
              │             │              │
           无限等待        不等待          等一段时间
                                           │
                                      try_acquire_until()
                                           │
                                      等到 deadline
                                      

```c
#include <semaphore>
#include <iostream>
#include <thread>
#include <chrono>
#include <vector>

using namespace std::chrono_literals;

int main()
{
    std::counting_semaphore<3> sem(3);

    auto worker = ([&](int id) {
        if (sem.try_acquire_for(4s))
        {
            std::cout << id << " acquired\n";
            std::this_thread::sleep_for(1s);
            sem.release();
        }
        else
        {
            std::cout << id << " timeout\n";
        }

        });

    std::vector<std::jthread> threads;
    threads.reserve(6);

    for (int i = 0; i < 5; i++)
    {
        threads.emplace_back(worker, i);
    }

}
```

