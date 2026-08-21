I have enough material from the proposal text and the committee discussion. Here is the full explanation.

## 背景：`std::barrier` 是什么

`std::barrier`（C++20 引入）是一种线程同步原语，用于让一组线程在某个执行阶段结束时互相等待。它的使用模型是：N 个线程各自完成一段工作，然后都调用 `arrive_and_wait()`，最后一个到达的线程触发"阶段完成步骤"（phase completion step），执行用户提供的 `CompletionFunction`，之后所有线程才被放行，进入下一个阶段：

```cpp
auto completion = []() noexcept {
    // 在所有线程完成当前阶段后执行，执行完才开放下一阶段
    swap_buffers();
};

std::barrier sync(thread_count, completion);

// 每个线程：
void worker() {
    while (true) {
        do_phase_work();
        sync.arrive_and_wait();   // 等待全部到齐，completion 执行完，再继续
    }
}
```

## C++20 原规范的问题

C++20 的规范在措辞上有一个关键约束：`CompletionFunction` 必须在"到达屏障的某个线程上"执行。这句话翻译成实现层面，含义是：`completion` 函数运行时所在的线程，必须是参与了本阶段 `arrive` 调用的线程之一。

这个约束对 CPU 上的实现来说完全自然——某个线程是最后到达的，它顺手执行 completion，然后唤醒其他线程，实现简单高效。但 这个规范无意中迫使所有实现都把 `CompletionFunction` 运行在"最后到达屏障的那个线程"上，从而阻止了 `std::barrier` 借助硬件加速的线程同步机制。

最直接的受害者是 GPU 和 NVIDIA 的 CUDA 架构。GPU 上的线程同步在硬件层面是由专用单元完成的，"哪个线程最后到达"这个概念对 GPU 调度模型来说本身就是模糊的。对于 GPU 这类架构，期望能放宽原始规定，允许 completion handler 运行在任意线程上，而不一定是到达屏障的线程之一——这样做的理由是为了局部性（locality of reference），给实现更大的自由度来选择在哪里运行 completion。

## P2588 的修复：放宽线程归属要求

P2588 的目标是找到一个"甜点"规范——既保留应用程序需要的功能保证，又允许高效实现。

修改后的规范把要求从"必须运行在某个到达的线程上"改成：`CompletionFunction` 在"阶段完成期间"执行，但不再要求执行线程必须是参与了 `arrive` 的线程。

这实际上给实现两种合法选择：

```
选择 A（CPU 实现的自然行为）：
  Thread-7 是最后到达的
  → Thread-7 直接执行 completion()
  → Thread-7 唤醒其他线程

选择 B（GPU / 硬件屏障的需求）：
  硬件屏障单元检测到所有线程到达
  → 由调度单元或其他辅助线程执行 completion()
  → 所有工作线程被放行
```

两种选择在放行时序上的保证完全一致——所有在 `arrive_and_wait()` 上等待的线程，在 `completion()` 执行完毕之前不会被放行。这个保证没有被削弱。

## 被主动拒绝的那个放宽

委员会在讨论中曾考虑过更激进的放宽：允许 completion 运行在一个全新的线程上（"or it is a new thread"）。这个表述最终被删去。

原因是线程本地存储（thread-local storage）的问题。如果 completion 运行在一个全新线程上，那个线程不持有任何之前到达线程写入的 TLS 数据，`thread_local` 变量的值完全不可预测。反对意见认为，如果 completion handler 依赖线程本地存储，这种行为会非常令人意外。最终保留的规定是：执行 completion 的线程，必须是对应同步阶段内某个 `arrive` 调用完成后仍然存活的线程——不可以是凭空冒出来的新线程。

## 对代码的影响

对于只在 CPU 上运行的代码，这个变更几乎没有可见影响——libstdc++ 和 libc++ 本来就没有严格执行"必须是最后到达的线程"这个约束，MSVC STL 执行了但也随之更新。feature-test macro 从 `202011L` 更新到 `202302L`：

```cpp
#if __cpp_lib_barrier >= 202302L
// P2588 语义：completion 可运行在任意参与线程上，不限于最后到达者
#endif
```

实际写 `CompletionFunction` 时，这个变更意味着一个过去可以侥幸成立的假设现在明确不再成立：

```cpp
// 危险假设（即便在 C++20 时代也是脆弱的，C++23 后明确不成立）：
// "completion 一定运行在 thread-7 上，所以可以访问 thread-7 的 TLS"

// 正确写法：completion 内只访问对所有参与线程都可见的共享状态，
// 不依赖任何特定线程的 TLS 内容。
auto safe_completion = []() noexcept {
    global_result.store(compute_shared_result());   // 共享状态，安全
    // my_thread_local_cache = ...                  // 危险，不要这样做
};
```

总结一句话：这个 DR 的本质是把 `std::barrier` 从一个"意外地只能在 CPU 上高效实现"的规范，修正为一个能被 GPU、硬件屏障单元等异构计算环境合理实现的规范，代价是明确禁止了 completion 函数依赖特定线程 TLS 的写法——而这种写法本来就是危险的。