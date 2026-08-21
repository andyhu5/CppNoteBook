#### std::ranges::range_adaptor_closure

 是 C++23 Ranges 库新增的一个**辅助基类（helper base class）**，用于让**用户自定义的 Range Adaptor**具有和标准库 `views::filter`、`views::transform` 一样的管道 (`|`) 组合能力。

它来自头文件：

```cpp
#include <ranges>
```

## 为什么需要它？

假设我们自己写一个 Range Adaptor：

```cpp
auto twice = [](auto&& r) {
    return r | std::views::transform([](int x) { return x * 2; });
};
```

它可以这样使用：

```cpp
twice(v);
```

但是不能这样：

```cpp
v | twice;          // ❌
```

也不能继续组合：

```cpp
views::filter(...)
| twice
| views::reverse;   // ❌
```

原因是普通 lambda 并不是一个 **Range Adaptor Closure Object**。

标准库中的

```cpp
std::views::filter(pred)
std::views::transform(func)
std::views::reverse
```

实际上都是 Closure Object，因此支持：

```cpp
range
| views::filter(...)
| views::transform(...)
| views::reverse;
```

C++23 提供 `range_adaptor_closure`，就是为了让用户自己的 adaptor 也拥有同样能力。

------

# Range Adaptor Closure 是什么？

标准规定，一个继承自

```cpp
std::ranges::range_adaptor_closure<T>
```

的类型：

```cpp
struct MyAdaptor
    : std::ranges::range_adaptor_closure<MyAdaptor>
{
    ...
};
```

自动获得：

```cpp
range | my_adaptor
```

以及

```cpp
adaptor1 | adaptor2
```

这样的能力。

它实际上提供了两个重要的 `operator|`。

概念上类似：

```cpp
range | adaptor
```

等价于

```cpp
adaptor(range)
```

而

```cpp
adaptor1 | adaptor2
```

则生成一个新的 Closure：

```
range
 ↓
adaptor1
 ↓
adaptor2
```

因此能够无限组合。

------

# 最简单例子

例如定义一个 reverse 两次（虽然没意义）：

```cpp
#include <ranges>
#include <vector>
#include <iostream>

struct twice_reverse
    : std::ranges::range_adaptor_closure<twice_reverse>
{
    template<std::ranges::viewable_range R>
    auto operator()(R&& r) const
    {
        return std::forward<R>(r)
            | std::views::reverse
            | std::views::transform([](auto&& x)->int& {
                x= x + 100;
				return x;
                });
    }
};

int main()
{
    std::vector v{ 1,2,3 };
    for (auto &i : v | twice_reverse{})
        std::cout << i << ' ';
}
```

输出

```
1 2 3
```

这里

```cpp
v | twice_reverse{}
```

实际上就是

```cpp
twice_reverse{}(v)
```

------

# 更实用的例子：乘以 2

```cpp
#include <ranges>
#include <vector>
#include <iostream>


struct multiply2
    : std::ranges::range_adaptor_closure<multiply2>
{
    template<std::ranges::viewable_range R>
    auto operator()(R&& r) const
    {
        return std::forward<R>(r)
            | std::views::transform(
                [](int x) { return x * 2; });
    }
};

int main()
{
    std::vector v{ 1,2,3 };
    for (int x : v | multiply2{})
        std::cout << x << ' ';
}
```

输出

```
2 4 6
```

------

# Closure 可以继续组合

最大的优势就是可以继续组合：

```cpp
auto pipeline =
        multiply2{}
    | std::views::reverse;
```

以后直接：

```cpp
v | pipeline;
```

等价于

```cpp
v
| multiply2{}
| std::views::reverse;
```

甚至可以：

```cpp
auto p =
      std::views::filter(...)
    | multiply2{}
    | std::views::take(5)
    | std::views::reverse;
```

形成新的 Closure。

------

# 带参数的 Adaptor

很多 View 都带参数，例如：

```cpp
views::drop(5)
```

自己的 adaptor 也可以这样写：

```cpp
struct multiply
    : std::ranges::range_adaptor_closure<multiply>
{
    int factor;

    explicit multiply(int n)
        : factor(n) {}

    template<std::ranges::viewable_range R>
    auto operator()(R&& r) const
    {
        return std::forward<R>(r)
            | std::views::transform(
                [k=factor](int x)
                {
                    return x * k;
                });
    }
};
```

使用：

```cpp
v | multiply(10);
```

结果

```
10 20 30
```

和标准库

```cpp
views::drop(5)
views::take(10)
views::stride(2)
```

的设计方式一致。

------

# 与 `views::transform` 的关系

实际上标准库中的：

```cpp
views::filter(pred)

views::transform(fun)

views::drop(n)

views::take(n)
```

返回的都是 **Range Adaptor Closure Object**。

因此：

```cpp
views::filter(...)
```

不是 View。

而是：

```
Closure
```

真正执行时：

```cpp
range
|
views::filter(...)
```

才产生：

```
filter_view
```

因此可以理解成：

```
filter(pred)
        │
        ▼
 Closure Object
        │
        ▼
operator|()
        │
        ▼
filter_view
```

------

# `range_adaptor_closure` 做了什么？

它本身几乎没有数据成员，也不会创建 View。它只是为派生类提供标准化的管道接口，包括：

- 支持 `range | adaptor`
- 支持 `adaptor1 | adaptor2`
- 自动生成组合 Closure
- 保证与标准 Views 的组合行为一致
- 避免用户手写复杂的 `operator|`

因此它只是一个 **CRTP（Curiously Recurring Template Pattern）辅助基类**。

------

## 与 C++20 的区别

C++20 中，标准库内部也有类似机制，但没有公开给用户。实现自定义可组合的 Range Adaptor 通常需要手写 `operator|`，代码复杂且与不同标准库实现耦合。

C++23 将这一能力标准化，公开了 `std::ranges::range_adaptor_closure`，使用户定义的 Adaptor 能与标准 `views::*` 无缝组合，获得与标准库相同的管道语法和组合能力。

## 总结

| C++20                                       | C++23                         |
| ------------------------------------------- | ----------------------------- |
| 自定义 Range Adaptor 需要自己实现 `operator | `                             |
| 很难与标准 `views` 无缝组合                 | 可直接与所有标准 `views` 组合 |
| 标准库内部机制未公开                        | 公开标准化的 CRTP 基类        |
| 代码量较大                                  | 只需继承并实现 `operator()`   |

一句话概括：`std::ranges::range_adaptor_closure` 是 C++23 为**程序员自定义 Range Adaptor**提供的标准基类，使其具备与 `std::views::filter`、`std::views::transform` 等标准视图适配器完全一致的 `|` 管道和组合能力。