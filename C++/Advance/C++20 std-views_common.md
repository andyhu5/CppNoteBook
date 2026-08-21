`std::views::common` 是 C++20 引入的 Range Adapter（不是 C++23 才有），它的作用可以概括为一句话：

> **将一个 Range 转换成 `common_range`，使 `begin()` 和 `end()` 返回相同的类型。**

在你的例子中，它并不是为了插入 `0`，而是为了**让结果可以方便地与传统 STL 算法和容器构造函数协同工作**。

------

## 什么是 `common_range`？

传统 STL 容器：

```cpp
std::vector
std::string
std::list
```

都有：

```cpp
begin() -> iterator
end()   -> iterator
```

例如：

```cpp
std::vector<int> v;

static_assert(
    std::same_as<
        decltype(v.begin()),
        decltype(v.end())
    >
);
```

两者类型一样。

这就是：

```text
common_range
```

------

## 但是 Ranges 不一样

很多 View：

```cpp
std::views::filter
std::views::take_while
std::views::join
std::views::join_with
```

为了效率，

它们通常：

```text
begin()

↓

iterator

end()

↓

sentinel
```

注意：

```text
iterator ≠ sentinel
```

例如：

```
iterator
    ↓
1 2 3 4 5

            ↑
         sentinel
```

sentinel（哨兵）只是：

> "结束标志"

它不是 iterator。

------

## 为什么这样设计？

例如：

```cpp
std::views::take_while(...)
```

结束位置：

不是固定的。

而是：

```cpp
predicate == false
```

因此：

```cpp
end()
```

没必要构造一个真正 iterator。

返回：

```text
sentinel
```

效率更高。

------

# `views::common`

它做的事情就是：

把：

```text
iterator

sentinel
```

包装成：

```text
iterator

iterator
```

于是：

```cpp
begin()

↓

iterator

end()

↓

iterator
```

成为：

```text
common_range
```

------

# 为什么你的代码需要它？

你的代码：

```cpp
auto flattened =
    data
    | std::views::join_with(0)
    | std::views::common;
```

下面：

```cpp
std::vector<int> flat(
    flattened.begin(),
    flattened.end());
```

这个构造函数来自：

```cpp
vector(first,last)
```

历史上要求：

```text
first

↓

InputIterator

last

↓

InputIterator
```

而：

```cpp
join_with
```

返回的是：

```text
iterator

+

sentinel
```

有些 STL 实现无法匹配。

所以：

```cpp
views::common
```

把：

```text
sentinel

↓

iterator
```

于是：

```cpp
vector(...)
```

可以正常工作。

------

## 一个更简单的例子

例如：

```cpp
std::vector<int> v{1,2,3,4,5};

auto r =
    v
    | std::views::filter(
        [](int x)
        {
            return x % 2;
        });
```

这里：

```cpp
decltype(r.begin())
```

不是：

```cpp
decltype(r.end())
```

所以：

```cpp
std::same_as<
decltype(r.begin()),
decltype(r.end())
>
```

得到：

```text
false
```

如果：

```cpp
auto c =
    r
    | std::views::common;
```

现在：

```text
begin()

↓

iterator

end()

↓

iterator
```

于是：

```cpp
same_as
```

变成：

```text
true
```

------

## 如何验证？

可以：

```cpp
#include <ranges>
#include <vector>
#include <type_traits>

std::vector<int> v{1,2,3};

auto r =
    v
    | std::views::filter(
        [](int x){ return x > 1; });

static_assert(
    !std::same_as<
        decltype(r.begin()),
        decltype(r.end())
    >
);

auto c =
    r
    | std::views::common;

static_assert(
    std::same_as<
        decltype(c.begin()),
        decltype(c.end())
    >
);
```

------

# `join_with` 为什么经常配 `common`？

例如：

```cpp
join
join_with
split
lazy_split
```

这些 View 都可能返回：

```text
iterator + sentinel
```

因此很多示例都会写：

```cpp
| std::views::common
```

就是为了：

让：

```cpp
begin()

end()
```

具有相同类型。

------

# 现代 STL 还需要吗？

其实：

```cpp
std::vector<int> flat(
    flattened.begin(),
    flattened.end());
```

在 **C++23 的大多数新版本 GCC / Clang / MSVC STL** 中，

很多情况下：

**即使没有 `views::common` 也能编译。**

因为：

STL 已经支持：

```text
iterator

+

sentinel
```

或者：

```cpp
std::ranges::begin
std::ranges::end
```

相关接口。

所以：

```cpp
views::common
```

更多是一种：

> **兼容旧接口**
>
> **保证得到 common_range**
>
> **方便与传统 STL 交互**

------

## 总结

| View                        | begin()  | end()    |
| --------------------------- | -------- | -------- |
| `std::vector`               | iterator | iterator |
| `filter_view`               | iterator | sentinel |
| `join_view`                 | iterator | sentinel |
| `join_with_view`            | iterator | sentinel |
| `filter | views::common`    | iterator | iterator |
| `join_with | views::common` | iterator | iterator |

**一句话概括：** `std::views::common` 的作用不是改变数据内容，而是将一个 **`iterator` 与 `sentinel` 类型不同的 Range** 包装成 **`begin()` 和 `end()` 都返回同一迭代器类型的 `common_range`**。这样可以更方便地与传统 STL 接口（如部分容器构造函数、旧算法或依赖同类型迭代器的代码）进行互操作。