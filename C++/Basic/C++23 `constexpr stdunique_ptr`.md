# C++23：`constexpr std::unique_ptr` 解释

C++23 通过提案 P2273R3（"Making `std::unique_ptr` constexpr"）让 `std::unique_ptr` 的构造函数、析构函数、以及大部分成员函数都变成了 `constexpr`，使得它可以在**编译期常量表达式**中使用。这是 C++20 引入"允许在 constexpr 中做动态内存分配"（`std::allocator` 的 constexpr 化）之后，智能指针生态的自然延续。

## 1. 前置基础：为什么现在才能做到

`unique_ptr` 内部本质上就是"管理一个通过 `new`/`delete` 分配的指针"。要让它在编译期可用，前提是 `new`/`delete` 本身在编译期可用——这正是 C++20 的 P0784（"更多 constexpr 容器"相关工作）所铺垫的：C++20 已经允许在 `constexpr` 函数中做**堆内存分配**，只要这块内存**在同一个常量求值过程结束前被释放**（不能"泄漏"到运行时）。

`unique_ptr` 恰好完美契合这个约束——它的整个设计理念就是"生命周期结束时自动释放"，天然满足"分配和释放都在同一个常量求值内完成"的要求，所以把它 constexpr 化是水到渠成的一步。

## 2. 基本用法示例

```cpp
#include <memory>

constexpr int compute() {
    std::unique_ptr<int> p = std::make_unique<int>(42);
    *p += 8;
    return *p;   // p 在函数结束时自动析构，内存在编译期被"释放"
}

static_assert(compute() == 50);  // C++23: 编译期求值通过！
```

在 C++20 里，上面这段代码是无法通过 `static_assert` 的（`unique_ptr` 的构造/析构函数不是 `constexpr`）；C++23 之后可以。

## 3. 哪些部分变成了 `constexpr`

- 默认构造函数、接管指针的构造函数、移动构造/赋值
- 析构函数（这个尤其关键，没有 constexpr 析构函数，编译期就没法正确"归还"分配的内存）
- `operator*`、`operator->`、`get()`、`release()`、`reset()`、`swap()` 等常用成员函数
- `std::make_unique`、`std::make_unique_for_overwrite` 也相应变成 `constexpr`

```cpp
constexpr bool test_reset() {
    auto p = std::make_unique<int>(1);
    p.reset(new int(2));   // constexpr 里也能重新分配/释放
    return *p == 2;
}
static_assert(test_reset());
```

## 4. 数组版本 `unique_ptr<T[]>` 也支持

```cpp
constexpr int sum_array() {
    auto arr = std::make_unique<int[]>(5);
    for (int i = 0; i < 5; ++i) arr[i] = i;
    int total = 0;
    for (int i = 0; i < 5; ++i) total += arr[i];
    return total;
}
static_assert(sum_array() == 10);
```

## 5. 自定义 deleter 的限制

如果你用了自定义删除器（`unique_ptr<T, Deleter>`），要在 constexpr 上下文里工作，**这个 deleter 本身也必须是 constexpr 可调用的**：

```cpp
struct MyDeleter {
    constexpr void operator()(int* p) const { delete p; }  // 必须是 constexpr
};

constexpr bool test_custom_deleter() {
    std::unique_ptr<int, MyDeleter> p(new int(5));
    return *p == 5;
}
static_assert(test_custom_deleter());
```

如果 deleter 不是 constexpr 可调用的（比如内部调用了某些非 constexpr 的 C 库函数），那这个 `unique_ptr` 实例就依然**不能**在编译期常量表达式里使用——这不是"部分退化"，而是那次具体的常量求值会直接失败。

## 6. 和 `shared_ptr` 的对比：为什么只有 `unique_ptr`

值得注意的是，**`std::shared_ptr` 在 C++23 里并没有变成 constexpr**。原因在于 `shared_ptr` 内部依赖**控制块**（引用计数、弱引用计数）以及可能的**类型擦除的 deleter**，实现上通常涉及更复杂的内存布局和原子操作（用于线程安全的引用计数），这些机制目前还没有被纳入 constexpr 求值的支持范围。`unique_ptr` 因为没有引用计数、没有共享状态，结构简单得多，所以率先完成了 constexpr 化。

## 7. 实际意义：这解决了什么真实痛点

在此之前，如果你想在编译期构建一棵树、链表，或者做一些需要动态分配的元编程/编译期计算，通常只能：

- 用固定大小数组模拟（浪费空间，且大小要提前知道）
- 用 `std::array` 之类静态容器绕过动态分配
- 完全放弃在编译期做这类计算，退回运行时

现在有了 constexpr `unique_ptr`（以及配套的 constexpr `vector`、`string` 等 C++20/23 陆续 constexpr 化的容器），可以更自然地在编译期写"看起来和运行时一样"的、涉及动态内存管理的算法，比如编译期构建并遍历一棵动态大小的树：

```cpp
struct Node {
    int value;
    std::unique_ptr<Node> left, right;
};

constexpr int build_and_sum() {
    auto root = std::make_unique<Node>(1, 
        std::make_unique<Node>(2, nullptr, nullptr),
        std::make_unique<Node>(3, nullptr, nullptr));
    return root->value + root->left->value + root->right->value;
}
static_assert(build_and_sum() == 6);
```

## 8. 一句话总结

**C++23 让 `std::unique_ptr`（包括它的构造、析构、`make_unique` 等常用操作）可以在 `constexpr` 上下文中使用，前提是自定义 deleter（若有）也必须是 constexpr 可调用的；这依赖于 C++20 已经允许的"编译期内堆分配必须在同一常量求值内释放"的规则，`unique_ptr` 天然满足这一约束，而结构更复杂、依赖共享控制块的 `shared_ptr` 则暂未获得同样的支持。**