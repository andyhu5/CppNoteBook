# condition_variable

```C++
std::jthread reader1([]() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });
    int value = v.load(std::memory_order_acquire);
    std::cout << value << std::endl;
});
```



在 cv.wait(lock, [] { return ready; }); 

中，[] { return ready; } 是一个谓词（predicate），用于指定条件变量 std::condition_variable 的等待条件。

以下是为什么需要 return ready 的详细解释：

1. **条件变量的等待机制**

std::condition_variable::wait 函数的第二个参数是一个谓词，用于检查是否满足继续执行的条件。它的作用是：

- **避免虚假唤醒（spurious wakeups）**：线程可能在没有收到 notify_one() 或 notify_all() 的情况下被唤醒（这是 C++ 标准库的实现特性）。谓词确保线程只在条件真正满足时继续执行。
- **检查条件**：在等待之前和每次唤醒后，wait 会调用谓词（这里是 [] { return ready; }）。如果谓词返回 false，线程继续等待；如果返回 true，线程继续执行。
- **为什么 return ready？**

- ready 是一个布尔变量，表示某个条件是否已满足（例如，writer 线程已经设置了 v 并准备好让 reader 线程读取）。
- [] { return ready; } 是一个 lambda 表达式，告诉 wait 函数检查 ready 的值：
  - 如果 ready == true，谓词返回 true，wait 结束，线程继续执行。
  - 如果 ready == false，谓词返回 false，线程继续等待（重新进入阻塞状态）。
- 具体来说，ready 在 writer 线程中被设置为 true（通常伴随着 cv.notify_all()），表示 reader 线程可以安全地读取 v 的值。
- **为什么不直接用 cv.wait(lock);？**

如果直接调用 cv.wait(lock);，没有谓词，wait 会在收到 notify_one() 或 notify_all() 时无条件唤醒。但由于虚假唤醒的存在，线程可能在 ready 尚未为 true 时被唤醒，导致逻辑错误（例如，读取未初始化的 v）。使用谓词 [] { return ready; } 确保线程只在 ready == true 时继续执行。

4. **等价代码**

cv.wait(lock, [] { return ready; }); 大致等价于以下显式循环：

cpp

```cpp
while (!ready) {
    cv.wait(lock);
}
```

但使用谓词形式更简洁，且由 std::condition_variable 内部优化处理，避免了手动循环的复杂性。

5. **为什么用 lambda 表达式？**

- Lambda 表达式 [] { return ready; } 提供了一种灵活的方式来定义检查条件。捕获列表 [] 表示不需要捕获外部变量（因为 ready 是全局变量）。如果条件更复杂（例如，检查多个变量），可以在 lambda 中编写更复杂的逻辑：

  cpp

  ```cpp
  cv.wait(lock, [] { return ready && some_other_condition; });
  ```

总结

return ready 是为了告诉 std::condition_variable::wait 只有在 ready == true 时才停止等待。这是确保线程同步正确、避免虚假唤醒的关键机制。如果没有谓词，线程可能在条件未满足时过早继续执行，导致未定义行为或逻辑错误。