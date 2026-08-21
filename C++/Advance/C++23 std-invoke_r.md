`std::invoke_r` 是 **C++23** 在 `<functional>` 中新增的一个工具函数，来源于提案 **P2136R3**。

它可以理解为：

> **带有返回值类型约束的 `std::invoke`。**

也就是说：

- `std::invoke`：调用任意 Callable。
- `std::invoke_r<R>`：调用任意 Callable，并将结果按 `R` 返回。

------

## 为什么需要 `std::invoke`？

先回顾 `std::invoke`。

它统一了所有可调用对象（Callable）的调用方式，包括：

- 普通函数
- Lambda
- 函数对象
- 成员函数指针
- 成员变量指针

例如：

```cpp
std::invoke(f, args...);
```

内部会自动判断：

```cpp
f(args...)

obj.*pmf(args...)

(ptr->*pmf)(args...)

obj.*member
```

等等。

因此：

```cpp
std::invoke
```

已经成为标准库内部调用 Callable 的统一入口。

------

## `std::invoke` 的不足

假设：

```cpp
int foo()
{
    return 10;
}
```

调用：

```cpp
auto x = std::invoke(foo);
```

返回类型：

```text
int
```

如果希望：

```cpp
double x = foo();
```

只能依赖：

```cpp
double x = std::invoke(foo);
```

发生隐式转换。

如果写模板：

```cpp
template<class F>
bool call(F&& f)
{
    return std::invoke(std::forward<F>(f));
}
```

虽然最终能返回 `bool`，但函数本身并没有明确要求 **Callable 的结果必须可转换为 `bool`**。

------

## C++23：`std::invoke_r`

于是新增：

```cpp
#include <functional>

std::invoke_r<R>(callable, args...);
```

它表示：

> 调用 Callable，并返回 **R**。

如果结果不能转换为 `R`，则编译失败。

------

## 函数原型

大致为：

```cpp
template<class R, class F, class... Args>
constexpr R invoke_r(F&& f, Args&&... args);
```

其中：

```text
R
```

是希望得到的返回类型。

------

## 一个简单例子

```cpp
#include <functional>
#include <iostream>

int foo()
{
    return 42;
}

int main()
{
    double x = std::invoke_r<double>(foo);

    std::cout << x;
}
```

输出：

```text
42
```

实际上：

```text
int

↓

double
```

进行了转换。

------

## 返回 bool

例如：

```cpp
int foo()
{
    return 100;
}
```

调用：

```cpp
bool b = std::invoke_r<bool>(foo);
```

得到：

```text
true
```

因为：

```text
100

↓

bool
```

转换成功。

------

## 返回 void

这是 `invoke_r` 最有价值的地方之一。

例如：

```cpp
int foo()
{
    return 123;
}
```

可以：

```cpp
std::invoke_r<void>(foo);
```

即：

```text
调用 foo()

↓

忽略返回值
```

以前：

```cpp
std::invoke(foo);
```

返回：

```text
int
```

只能：

```cpp
(void)std::invoke(foo);
```

现在更加自然。

------

## Lambda

```cpp
auto f = []{
    return 3.14;
};

int x =
    std::invoke_r<int>(f);
```

得到：

```text
3
```

进行了：

```text
double

↓

int
```

转换。

------

## 成员函数

```cpp
struct S
{
    int get() const
    {
        return 10;
    }
};

S s;

double d =
    std::invoke_r<double>(&S::get, s);
```

得到：

```text
10.0
```

------

## 不可转换时

例如：

```cpp
struct A {};

A foo();
```

调用：

```cpp
std::invoke_r<int>(foo);
```

编译失败。

因为：

```text
A

↓

int
```

不存在转换。

------

## 与 `std::invoke` 的区别

| 功能          | `std::invoke` | `std::invoke_r` |
| ------------- | ------------- | --------------- |
| 调用 Callable | ✔             | ✔               |
| 指定返回类型  | ❌             | ✔               |
| 返回值转换    | 自动推导      | 转换成 `R`      |
| 支持 `void`   | 间接          | ✔               |

------

## 为什么 C++23 才加入？

很多标准库算法都需要：

```cpp
invoke

↓

转换成指定类型
```

例如：

```cpp
bool pred(...)
```

实际上：

```cpp
invoke(pred)

↓

bool
```

Ranges 库中有大量：

```cpp
static_cast<R>(
    std::invoke(...)
)
```

或者：

```cpp
if constexpr(std::is_void_v<R>)
    ...
else
    ...
```

每个地方都要自己写。

因此标准委员会抽象出了：

```cpp
std::invoke_r
```

统一完成这些工作。

------

## 一个典型实现（简化）

大致等价于：

```cpp
template<class R, class F, class... Args>
constexpr R invoke_r(F&& f, Args&&... args)
{
    if constexpr (std::is_void_v<R>)
    {
        std::invoke(std::forward<F>(f),
                    std::forward<Args>(args)...);
    }
    else
    {
        return static_cast<R>(
            std::invoke(std::forward<F>(f),
                        std::forward<Args>(args)...));
    }
}
```

真正的标准实现还会使用 `std::is_invocable_r_v` 等约束，并正确处理异常说明（`noexcept`），但核心思想就是如此。

------

## 与 `std::is_invocable_r` 的关系

C++17 已经有：

```cpp
std::is_invocable_r_v<R, F, Args...>
```

用于判断：

```text
invoke(f,args...)

↓

是否可以转换成 R
```

C++23 又提供：

```cpp
std::invoke_r<R>(...)
```

真正执行调用。

二者形成完整配套：

| 类型萃取                | 调用函数        |
| ----------------------- | --------------- |
| `std::is_invocable_v`   | `std::invoke`   |
| `std::is_invocable_r_v` | `std::invoke_r` |

------

## 总结

| 特性                       | `std::invoke`  | `std::invoke_r` |
| -------------------------- | -------------- | --------------- |
| C++版本                    | C++17          | C++23           |
| 头文件                     | `<functional>` | `<functional>`  |
| 调用任意 Callable          | ✔              | ✔               |
| 指定返回类型               | ❌              | ✔               |
| 自动转换返回值             | 推导           | 转换为 `R`      |
| 支持忽略返回值（`R=void`） | 间接           | ✔               |

**一句话概括：** `std::invoke_r<R>` 是 C++23 对 `std::invoke` 的扩展，它在统一调用各种 Callable 的基础上，**要求返回值能够转换为指定类型 `R`**。如果 `R` 为 `void`，则调用后直接丢弃返回值；如果不能转换为 `R`，则编译失败。它使泛型代码中对返回类型的约束和处理更加简洁、统一。