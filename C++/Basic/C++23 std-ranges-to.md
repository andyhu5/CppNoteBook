`std::ranges::to` 是 C++23 Ranges 最实用的新功能之一，它的作用就是：

> **把一个 Range（通常是 View）转换成一个具体容器（Container）。**

头文件：

```cpp
#include <ranges>
```

语法：

```cpp
std::ranges::to<Container>(range)
```

或者作为管道：

```cpp
range | std::ranges::to<Container>()
```

它解决了 C++20 中最大的痛点之一：**View 很方便，但最终如何优雅地变成 `std::vector`、`std::string`、`std::set` 等容器非常麻烦。**

------

## C++20 的问题

例如：

```cpp
std::vector v{1,2,3,4,5};

auto view =
    v
    | std::views::filter([](int x){ return x % 2; })
    | std::views::transform([](int x){ return x * 10; });
```

`view` 的类型实际上是：

```cpp
transform_view<
    filter_view<...>
>
```

如果想得到真正的 vector，只能写：

```cpp
std::vector<int> result(view.begin(), view.end());
```

或者

```cpp
std::vector<int> result;
std::ranges::copy(view, std::back_inserter(result));
```

既冗长又容易写错。

------

# C++23：std::ranges::to

有了 `to`：

```cpp
std::vector result =
    std::ranges::to<std::vector>(view);
```

或者

```cpp
auto result =
    std::ranges::to<std::vector>(view);
```

编译器能够推导元素类型。

------

# 管道写法（最推荐）

```cpp
auto result =
    v
    | std::views::filter([](int x)
        {
            return x % 2;
        })
    | std::views::transform([](int x)
        {
            return x * 10;
        })
    | std::ranges::to<std::vector>();
```

最终得到：

```cpp
std::vector<int>
```

这也是 C++23 推荐的写法。

------

# 转成 string

例如：

```cpp
std::string s = "abcdef";
```

过滤：

```cpp
auto str =
    s
    | std::views::filter([](char c)
        {
            return c != 'c';
        })
    | std::ranges::to<std::string>();
```

得到

```text
abdef
```

以前需要：

```cpp
std::string result(view.begin(), view.end());
```

现在一句即可。

------

# 转成 set

例如：

```cpp
std::vector v{4,1,5,3,2,3,4,5};
```

直接：

```cpp
auto s =
    v
    | std::ranges::to<std::set>();
```

结果：

```text
1 2 3 4 5
```

自动去重排序。

------

# 转成 unordered_set

```cpp
auto us =
    v
    | std::ranges::to<std::unordered_set>();
```

十分自然。

------

# 转成 deque

```cpp
auto d =
    view
    | std::ranges::to<std::deque>();
```

得到

```cpp
std::deque<int>
```

------

# 转成 list

```cpp
auto lst =
    view
    | std::ranges::to<std::list>();
```

得到

```cpp
std::list<int>
```

------

# 转成 map

例如：

```cpp
std::vector<std::pair<int,std::string>> v{
    {1,"A"},
    {2,"B"},
    {3,"C"}
};
```

可以直接：

```cpp
auto m =
    v
    | std::ranges::to<std::map>();
```

得到：

```cpp
std::map<int,std::string>
```

------

# 元素类型自动推导

例如：

```cpp
std::vector<int> v{1,2,3};
```

经过 transform：

```cpp
auto r =
    v
    | std::views::transform([](int x)
        {
            return x * 0.5;
        });
```

元素已经变成：

```cpp
double
```

于是：

```cpp
auto d =
    r
    | std::ranges::to<std::vector>();
```

得到：

```cpp
std::vector<double>
```

无需写：

```cpp
std::vector<double>
```

编译器自动推导。

------

# 指定容器元素类型

如果想指定：

```cpp
auto v =
    r
    | std::ranges::to<std::vector<int>>();
```

每个元素都会转换成：

```cpp
int
```

例如：

```cpp
1.2
2.7
```

变成：

```text
1
2
```

------

# 与 `std::vector(begin,end)` 的区别

例如：

```cpp
std::vector result(view.begin(), view.end());
```

只能用于支持迭代器构造函数的容器。

而

```cpp
std::ranges::to<std::vector>()
```

标准库内部会自动选择最佳方式：

- reserve（如果支持）
- append
- insert
- push_back
- emplace_back

因此对于不同容器都能工作。

------

# 实现思想

可以近似理解为：

```cpp
template<class C, std::ranges::input_range R>
C to(R&& r)
{
    C c;

    for (auto&& x : r)
        c.insert(c.end(), x);

    return c;
}
```

真正标准实现复杂得多，会根据容器能力优化，例如：

- 是否支持 `reserve()`
- 是否支持 `append_range()`（C++23）
- 是否支持 `insert_range()`
- 是否支持 `assign_range()`
- 是否支持 `push_back()`

因此效率通常优于手写循环。

------

## 与 C++23 其他容器接口的协同

`std::ranges::to` 与 C++23 新增的容器 Range 接口配合得很好，例如：

```cpp
std::vector<int> src{1,2,3};

std::list<int> lst;
lst.append_range(src);

std::set<int> s;
s.insert_range(src);
```

这些接口解决了**已有容器接收一个 Range**的问题，而 `std::ranges::to` 则解决了**从一个 Range 创建新容器**的问题。

## 总结

| 写法                 | C++20                             | C++23                                   |
| -------------------- | --------------------------------- | --------------------------------------- |
| View → `std::vector` | `std::vector(v.begin(), v.end())` | `std::ranges::to<std::vector>(v)`       |
| View → `std::string` | `std::string(v.begin(), v.end())` | `std::ranges::to<std::string>(v)`       |
| View → `std::set`    | 手动构造                          | `std::ranges::to<std::set>(v)`          |
| 管道风格             | 不支持                            | `view | std::ranges::to<std::vector>()` |

可以把 `std::ranges::to` 看作 **Ranges 管道的“终结器（terminal operation）”**：前面的 `filter`、`transform`、`take`、`drop` 等都只是惰性生成 View，而 `to` 才真正遍历整个 Range，并将结果物化（materialize）为一个具体容器。