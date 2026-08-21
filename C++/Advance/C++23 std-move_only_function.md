####  **`std::move_only_function`**

这是 **C++23** 新增的函数包装器，定义在 `<functional>` 中。

它可以理解为：

> **`std::function` 的 Move-Only 版本。**

最大的区别是：

- `std::function`：要求可复制（CopyConstructible）
- `std::move_only_function`：只要求可移动（MoveConstructible）

因此，它能够保存**不可复制（move-only）**的可调用对象，例如捕获了 `std::unique_ptr` 的 Lambda。

------

## 为什么需要它？

先看 `std::function` 的限制。

例如：

```cpp
#include <functional>
#include <memory>

auto f = [p = std::make_unique<int>(42)] {
    return *p;
};
```

这是一个合法的 Lambda。

但是：

```cpp
std::function<int()> func = std::move(f);
```

**编译失败。**

为什么？

因为：

Lambda 的成员包含

```cpp
std::unique_ptr<int>
```

于是 Lambda 自己就是

```cpp
CopyConstructible ❌
MoveConstructible ✔
```

而 `std::function` 要求：

```text
Stored callable must be CopyConstructible.
```

所以无法存储。

------

## `std::move_only_function`

C++23：

```cpp
#include <functional>
#include <memory>

std::move_only_function<int()> func =
    [p = std::make_unique<int>(42)] {
        return *p;
    };
```

现在可以正常工作。

因为：

```text
unique_ptr
      ↓
Lambda
      ↓
MoveConstructible
      ↓
move_only_function
```

整个链路都是 move-only。

------

## 一个完整例子

```cpp
#include <functional>
#include <iostream>
#include <memory>

int main()
{
    std::move_only_function<void()> f =
        [p = std::make_unique<int>(123)] {
            std::cout << *p << '\n';
        };

    f();
}
```

输出：

```text
123
```

------

## 可以移动，但不能复制

例如：

```cpp
std::move_only_function<void()> f = [] {};

auto g = std::move(f);
```

可以。

但是：

```cpp
auto g = f;
```

编译错误。

因为：

```text
Copy Constructor
=
deleted
```

------

## 与 `std::function` 对比

| 特性                    | `std::function` | `std::move_only_function` |
| ----------------------- | --------------- | ------------------------- |
| C++版本                 | C++11           | C++23                     |
| 可复制                  | ✔               | ❌                         |
| 可移动                  | ✔               | ✔                         |
| 可保存 move-only Lambda | ❌               | ✔                         |
| 可保存 `unique_ptr`     | ❌               | ✔                         |

------

## 支持更多 cv/ref/noexcept 组合

`std::move_only_function` 比 `std::function` 更现代。

例如可以表示：

```cpp
std::move_only_function<void() &>
std::move_only_function<void() &&>
std::move_only_function<int() const>
std::move_only_function<void() noexcept>
```

例如：

```cpp
std::move_only_function<void() noexcept> f =
    []() noexcept {};
```

调用对象必须也是 `noexcept`。

同样：

```cpp
std::move_only_function<void() const>
```

只能绑定：

```cpp
void operator()() const;
```

这种 callable。

这是 `std::function` 做不到的。

------

## 为什么 `std::function` 不直接修改？

理论上不能。

例如：

```cpp
std::function<void()> a = ...;

std::function<void()> b = a;
```

世界上已有大量代码依赖：

```text
std::function
=
Copyable
```

如果修改：

```text
Copyable
↓
Move-only
```

会破坏 ABI，也会导致大量旧代码无法编译。

因此标准委员会选择：

新增

```cpp
std::move_only_function
```

而不是修改

```cpp
std::function
```

------

## 与 `std::packaged_task` 的区别

很多人第一次看到它会想到：

```cpp
std::packaged_task
```

二者完全不同。

```cpp
std::packaged_task<int()>
```

主要用于：

- `std::future`
- `std::promise`
- 异步任务

它具有一次性执行（one-shot）的语义。

而：

```cpp
std::move_only_function<int()>
```

只是一个普通的函数包装器。

例如：

```cpp
f();
f();
f();
```

可以反复调用（前提是内部可调用对象支持）。

------

## 典型应用场景

### 1. 线程池

```cpp
class ThreadPool
{
    std::queue<std::move_only_function<void()>> tasks;
};
```

现在任务可以捕获：

```cpp
std::unique_ptr
std::fstream
std::mutex
socket
```

等不可复制资源。

------

### 2. 协程（Coroutine）

协程状态通常包含：

```cpp
unique_ptr
generator
```

因此：

```cpp
std::move_only_function
```

比

```cpp
std::function
```

更适合作为协程回调。

------

### 3. RAII 回调

例如：

```cpp
auto cleanup =
    [fd = std::move(file)] {
        close(fd);
    };
```

这种 Lambda 无法放进：

```cpp
std::function
```

但可以放进：

```cpp
std::move_only_function
```

------

## 总结

| 项目                                        | `std::function` | `std::move_only_function`         |
| ------------------------------------------- | --------------- | --------------------------------- |
| C++版本                                     | C++11           | C++23                             |
| 是否可复制                                  | ✔               | ❌                                 |
| 是否可移动                                  | ✔               | ✔                                 |
| 是否支持 move-only Callable                 | ❌               | ✔                                 |
| 是否支持 `unique_ptr` 捕获                  | ❌               | ✔                                 |
| 支持 `const` / `&` / `&&` / `noexcept` 限定 | 有限            | ✔ 完整支持                        |
| 典型用途                                    | 通用回调        | 线程池、协程、异步框架、RAII 回调 |

**一句话概括：** `std::move_only_function` 是 C++23 引入的 **Move-Only 多态函数包装器**。它继承了 `std::function` 的类型擦除（type erasure）能力，但去掉了“必须可复制”的限制，从而能够存储和传递捕获 `std::unique_ptr` 等独占资源的 Lambda 或其他仅可移动（move-only）的可调用对象。这使它更适合现代 C++ 中大量使用 RAII 和移动语义的设计。