`lambdas in unevaluated contexts`（**未求值上下文中的 Lambda**）是 **C++20** 引入的一项语言改进，来源于 WG21 提案 **P0315R4**。它允许在**未求值上下文（unevaluated context）**中直接编写 Lambda 表达式，而在 C++17 及之前这是不允许的。

这项改动看似很小，但对于模板元编程、概念（Concepts）、类型推导和泛型库开发非常有价值。

------

## 什么是 unevaluated context（未求值上下文）

未求值上下文是指**编译器只分析表达式的类型或语法，而不会真正执行表达式**。

常见的未求值上下文包括：

| 表达式                       | 是否求值 |
| ---------------------------- | -------- |
| `sizeof(expr)`               | ❌        |
| `decltype(expr)`             | ❌        |
| `noexcept(expr)`             | ❌        |
| `typeid(expr)`（非多态对象） | ❌        |
| `requires` 表达式（C++20）   | ❌        |

例如：

```cpp
sizeof(x++);
```

这里

```cpp
x++;
```

不会真正执行。

------

# C++17：Lambda 不允许出现在这些地方

例如

```cpp
decltype([]{});
```

C++17：

```
error: lambda-expression in unevaluated context
```

同样：

```cpp
sizeof([]{});
```

也非法。

原因是标准曾规定：

> Lambda expression shall not appear in an unevaluated operand.

------

# C++20：允许

C++20 删除了这个限制。

于是下面全部合法：

```cpp
decltype([]{});
sizeof([]{});
noexcept([]{});
```

例如：

```cpp
#include <type_traits>

using T = decltype([]{});

static_assert(std::is_class_v<T>);
```

可以正常编译。

------

# 示例1：decltype

以前：

```cpp
decltype([](int x)
{
    return x + 1;
});
```

非法。

现在：

```cpp
using Lambda = decltype([](int x)
{
    return x + 1;
});

Lambda f;

static_assert(f(3) == 4);
```

------

# 示例2：作为模板参数推导

例如：

```cpp
template<typename F>
void print_type(F);

int main()
{
    print_type(
        decltype([](int x)
        {
            return x * 2;
        }){}
    );
}
```

这里

```
decltype(lambda)
```

就是 lambda 的闭包类型（closure type）。

以前必须：

```cpp
auto f = [](int x)
{
    return x * 2;
};

print_type(f);
```

现在可以完全不创建变量。

------

# 示例3：sizeof

```cpp
constexpr auto size =
    sizeof([]{});
```

返回闭包对象大小。

例如：

```cpp
#include <iostream>

int main()
{
    std::cout << sizeof([]{}) << '\n';
}
```

通常输出

```
1
```

因为空闭包至少占一个字节。

------

# 示例4：noexcept

```cpp
constexpr bool b =
    noexcept([]{});
```

这里检查构造 lambda 是否会抛异常。

------

# 示例5：Concepts（最常见）

C++20 Concepts 中经常需要写：

```cpp
template<typename T>
concept Incrementable =
requires(T t)
{
    ++t;
};
```

如果需要一个辅助 Lambda，

以前必须：

```cpp
auto helper = [](auto x)
{
    return x + 1;
};
```

再引用它。

现在可以直接：

```cpp
using Helper =
decltype([](auto x)
{
    return x + 1;
});
```

无需污染命名空间。

------

# 示例6：Traits

例如：

```cpp
template<typename T>
struct wrapper
{
    using type =
        decltype([](T value)
        {
            return value;
        });
};
```

以前根本做不到。

------

# 为什么这很有用？

以前如果需要 Lambda 的类型：

```cpp
auto f = []{};
```

必须：

1. 创建变量
2. `decltype(f)`

例如：

```cpp
auto cmp = [](int a, int b)
{
    return a < b;
};

std::set<int, decltype(cmp)> s(cmp);
```

如果只想得到类型，这一步非常繁琐。

C++20：

```cpp
std::set<
    int,
    decltype([](int a, int b)
    {
        return a < b;
    })
> s;
```

直接获得比较器类型。

------

# 对泛型库的重要意义

这项改动对于 STL 实现者和泛型库作者帮助很大。

例如：

以前写 Traits：

```cpp
template<typename T>
using predicate_type =
decltype(predicate);
```

必须提前声明：

```cpp
inline constexpr auto predicate =
[](auto&& x)
{
    return x > 0;
};
```

现在：

```cpp
template<typename T>
using predicate_type =
decltype([](auto&& x)
{
    return x > 0;
});
```

无需任何额外对象。

------

# 与 `constexpr` 的结合

Lambda 本身就是 `constexpr`（满足条件时）。

例如：

```cpp
constexpr auto square =
[](int x)
{
    return x * x;
};

static_assert(square(5) == 25);
```

C++20 进一步允许：

```cpp
using Square =
decltype([](int x)
{
    return x * x;
});
```

既能在编译期推导类型，又无需定义变量。

------

# 总结

`lambdas in unevaluated contexts` 的核心就是**允许 Lambda 出现在不会真正执行表达式的上下文中**，使 Lambda 可以像普通类型表达式一样参与模板和类型系统。

主要收益包括：

- 可以直接使用 `decltype([]{...})` 获取 Lambda 闭包类型。
- 可以在 `sizeof`、`decltype`、`noexcept`、`requires` 等未求值上下文中编写 Lambda。
- 减少为了获取 Lambda 类型而引入的临时变量，使代码更简洁。
- 改善模板元编程、Concepts、类型萃取和泛型库的实现体验。

这项特性虽然不会改变 Lambda 的运行时行为，但显著提升了它们在编译期类型系统中的可用性，是 C++20 对模板和泛型编程能力的一项重要增强。