# C++23：`constexpr std::unique_ptr`（P2273R3）

## 前置条件：C++20 的 `constexpr` 动态内存分配

要理解 `constexpr unique_ptr` 是如何做到的，得先了解 C++20 引入的一项基础设施：**`constexpr new`/`delete`**（P0784R7）。C++20 允许在常量求值上下文中使用 `new`/`delete` 表达式动态分配、释放内存，但有一个关键限制：

> **通过 `constexpr new` 分配的内存，必须在同一次常量表达式求值结束之前，被完全 `delete` 掉，不能"泄漏"到运行期。**

也就是说，编译期分配的内存不可能变成运行期程序里的一块真实堆内存——它只能存在于"编译期计算的沙盒"里，用完就必须清理干净，否则这个常量表达式本身就是 ill-formed（编译错误）。

```cpp
constexpr int f() {
    int* p = new int(42); // constexpr new，合法
    int val = *p;
    delete p;              // 必须在同一常量求值内释放
    return val;
}
static_assert(f() == 42); // OK
```

## `unique_ptr` 天然契合这个模型

`std::unique_ptr` 的核心语义正好是：**独占所有权，析构时自动释放**——这正好完美匹配"必须在同一次求值内完全释放"这个约束。C++23（P2273R3）借助这个基础设施，把 `unique_ptr` 及其相关操作标记为 `constexpr`。

## 变成 `constexpr` 的成员/操作

- 构造函数（默认构造、从裸指针构造、移动构造等）
- 析构函数
- `operator=`（移动赋值）
- `reset()`
- `release()`
- `get()`
- `operator*()`、`operator->()`
- `operator bool()`
- `swap()`
- 非成员的 `std::make_unique`、`std::make_unique_for_overwrite`
- 比较运算符（`==`、`<=>` 等）

## 使用示例

### 基本用法

```cpp
constexpr int f() {
    auto p = std::make_unique<int>(42); // constexpr 分配
    int val = *p;
    return val;
} // p 在这里离开作用域，析构函数被调用，内存被 constexpr delete 释放
static_assert(f() == 42); // OK，整个过程完全在编译期完成
```

### 编译期构建、使用一个包含 `unique_ptr` 成员的数据结构

```cpp
struct Node {
    int value;
    std::unique_ptr<Node> next;
};

constexpr int sum_list() {
    std::unique_ptr<Node> head = std::make_unique<Node>(1, nullptr);
    head->next = std::make_unique<Node>(2, nullptr);
    head->next->next = std::make_unique<Node>(3, nullptr);

    int total = 0;
    for (Node* p = head.get(); p != nullptr; p = p->next.get()) {
        total += p->value;
    }
    return total;
    // head 离开作用域时，整个链表被递归析构，所有 constexpr 分配的内存被释放
}

static_assert(sum_list() == 6); // 编译期构建了一个链表、遍历求和、再销毁，全程在编译期完成
```

## 关键限制：不能把编译期分配的内存"带"到运行期

```cpp
constexpr std::unique_ptr<int> g() {
    return std::make_unique<int>(42); // ❌ 如果在 constexpr 上下文求值，
                                        //    返回的 unique_ptr 仍持有编译期分配的内存，
                                        //    这块内存无法合法地"跨越"到运行期继续存在
}

constexpr auto p = g(); // 编译错误：分配的内存必须在同一常量表达式内释放，不能被外泄
```

也就是说，`constexpr unique_ptr` 的典型使用模式是：**在一个 `constexpr`/`consteval` 函数内部，完整地"分配 → 使用 → 释放"整个生命周期**，最终返回的是**计算结果**（比如一个 `int`、一个不涉及动态分配的普通值类型），而不是把 `unique_ptr` 本身当作可以安全"逃逸"出编译期上下文的产物。

```cpp
constexpr int compute_with_heap() {
    auto data = std::make_unique<int[]>(10); // 编译期"堆"分配一个数组
    for (int i = 0; i < 10; ++i) data[i] = i * i;
    
    int sum = 0;
    for (int i = 0; i < 10; ++i) sum += data[i];
    return sum;
    // data 在函数结束时析构，内存被释放，只有 sum（一个普通 int）被返回
}

static_assert(compute_with_heap() == 285); // 0+1+4+9+...+81 = 285
```

## 和运行期使用完全一样，无需改代码

因为 `unique_ptr` 的接口没有变化，只是新增了 `constexpr` 修饰，所以同一份代码在运行期使用时行为和以前完全一致——`constexpr` 只是让**同一份实现**多了一种"也可以在编译期被求值"的能力，不需要为编译期/运行期分别维护两套逻辑（这也是 `constexpr` 这套机制一贯的设计哲学）。

```cpp
void runtime_usage() {
    auto p = std::make_unique<int>(100); // 运行期正常使用，语义、性能都不变
    *p += 1;
}
```

## 典型应用场景

1. **编译期构建复杂的树/链表/图结构做验证或生成**（比如编译期解析某种小型 DSL，构建 AST，再遍历求值/优化，最终产出一个编译期常量结果）。
2. **`consteval` 函数中需要临时的、动态大小的中间数据结构**，用完即弃，只关心最终计算结果。
3. **单元测试/`static_assert` 验证**：验证某个涉及动态内存管理的算法逻辑在各种输入下的正确性，可以直接在编译期跑一遍算法逻辑做断言，而不需要等到运行期。

## 小结

`constexpr std::unique_ptr` 的实现核心是复用了 C++20 就已经存在的 `constexpr` 动态内存分配基础设施（`constexpr new`/`delete`），并利用 `unique_ptr` "独占所有权、离开作用域自动释放"这一天然契合"内存必须在同一常量表达式内完全释放"约束的语义，让开发者可以在编译期**动态地分配、使用、释放内存**，而不仅仅局限于固定大小的 `std::array` 等静态结构。这是 C++23 一系列"扩大 `constexpr` 覆盖范围"改动（`bitset`、数学函数、`to_chars`/`from_chars` 等）里，涉及**动态内存管理**这一相对底层、也相对复杂的一块拼图。