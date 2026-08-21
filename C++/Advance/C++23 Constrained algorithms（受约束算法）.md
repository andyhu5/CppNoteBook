**Constrained algorithms（受约束算法）** 是 C++20 Ranges 引入的一类新算法，位于命名空间：

```cpp
std::ranges
```

它们对应于传统 STL 算法（`std::sort`、`std::find`、`std::copy` 等），但使用 **Concepts（概念）** 对模板参数进行约束，因此称为 **Constrained algorithms**。

简单来说：

> **Constrained algorithms = 使用 Concepts + Ranges 重新设计的 STL 算法。**

------

## 为什么叫 "Constrained"？

先看传统 STL。

例如 `std::sort`：

```cpp
std::sort(v.begin(), v.end());
```

模板大致是：

```cpp
template<class RandomIt>
void sort(RandomIt first, RandomIt last);
```

这里的 `RandomIt` 没有限制。

如果传入一个 `std::list` 的迭代器：

```cpp
std::list<int> lst{1,2,3};

std::sort(lst.begin(), lst.end()); // 编译失败
```

错误通常非常长，因为模板实例化到内部才发现：

```text
operator-
operator+
random access iterator required
...
```

真正的问题是：

> list 的迭代器不是 Random Access Iterator。

------

## C++20 的 Constrained Algorithm

C++20：

```cpp
std::ranges::sort(v);
```

其声明大致可以理解为：

```cpp
template<
    std::ranges::random_access_range R
>
constexpr auto sort(R&& r);
```

这里直接要求：

```cpp
R 必须满足 random_access_range
```

如果：

```cpp
std::list<int> lst;

std::ranges::sort(lst);
```

编译器立即指出：

```text
constraint not satisfied

random_access_range not satisfied
```

错误信息短得多，也更准确。

这就是 **Constrained** 的含义。

------

# 与传统算法的区别

例如排序：

传统 STL：

```cpp
std::sort(v.begin(), v.end());
```

Ranges：

```cpp
std::ranges::sort(v);
```

无需：

```cpp
.begin()
.end()
```

------

例如 find：

传统：

```cpp
auto it = std::find(v.begin(), v.end(), 5);
```

Ranges：

```cpp
auto it = std::ranges::find(v, 5);
```

更加自然。

------

## 支持 View

例如：

```cpp
auto odd =
    v
    | std::views::filter([](int x)
        {
            return x % 2;
        });
```

可以直接：

```cpp
auto it =
    std::ranges::find(odd, 7);
```

传统 STL：

```cpp
std::find(odd.begin(), odd.end(), 7);
```

虽然也能工作，但没有利用 Range 的接口。

------

# Projection（投影）

这是 Constrained Algorithms 最重要的新特性之一。

例如：

```cpp
struct Person
{
    std::string name;
    int age;
};

std::vector<Person> people;
```

找年龄：

传统：

```cpp
auto it =
    std::find_if(
        people.begin(),
        people.end(),
        [](const Person& p)
        {
            return p.age == 20;
        });
```

Ranges：

```cpp
auto it =
    std::ranges::find(
        people,
        20,
        &Person::age
    );
```

第三个参数：

```cpp
&Person::age
```

就是 Projection。

它表示：

```
Person
   │
   ▼
 age
   │
比较
```

无需写 Lambda。

------

排序同样如此：

```cpp
std::ranges::sort(
    people,
    {},
    &Person::age
);
```

表示：

```
按 age 排序
```

不用：

```cpp
[](auto const& a, auto const& b)
{
    return a.age < b.age;
}
```

------

# 返回值更合理

例如：

```cpp
std::ranges::copy(src, out);
```

返回：

```cpp
copy_result
```

包含：

```cpp
{
    in,
    out
}
```

而不是：

```cpp
OutputIterator
```

这样方便继续链式处理。

例如：

```cpp
auto r =
    std::ranges::copy(src, out);

r.in
r.out
```

------

# 更安全

例如：

```cpp
std::ranges::copy(a, b);
```

编译器会检查：

- input_range
- weakly_incrementable
- indirectly_copyable

是否满足。

以前：

```cpp
std::copy(...)
```

很多错误要进入模板深处才发现。

------

# 常见 Constrained Algorithms

几乎所有 STL 算法都有对应版本。

| STL              | Ranges                   |
| ---------------- | ------------------------ |
| `std::sort`      | `std::ranges::sort`      |
| `std::find`      | `std::ranges::find`      |
| `std::copy`      | `std::ranges::copy`      |
| `std::fill`      | `std::ranges::fill`      |
| `std::equal`     | `std::ranges::equal`     |
| `std::reverse`   | `std::ranges::reverse`   |
| `std::for_each`  | `std::ranges::for_each`  |
| `std::transform` | `std::ranges::transform` |
| `std::count`     | `std::ranges::count`     |
| `std::remove`    | `std::ranges::remove`    |

使用方式基本一致，只是支持直接传入 Range。

------

# 与 Views 的关系

很多人容易混淆这两者。

**Views（`std::views`）**：

- 惰性（Lazy）
- 不执行算法
- 只是生成新的 View

例如：

```cpp
auto r =
    v
    | std::views::filter(...)
    | std::views::transform(...);
```

此时没有遍历任何元素。

------

**Constrained Algorithms（`std::ranges`）**：

- 真正执行算法
- 遍历 Range
- 产生结果

例如：

```cpp
std::ranges::sort(v);

std::ranges::find(v, 5);

std::ranges::copy(v, out);
```

它们会立即执行。

------

## 总结

C++20 的 **Constrained Algorithms** 可以理解为 **现代化的 STL 算法**，主要改进包括：

- 使用 **Concepts** 对模板参数进行约束，编译错误更清晰。
- 支持直接接受 **Range**，无需手动传递 `begin()` 和 `end()`。
- 原生支持 **Views**，与 Ranges 生态无缝协作。
- 支持 **Projection**，许多场景无需编写比较或提取字段的 Lambda。
- 返回类型经过重新设计，更适合组合和链式处理。

因此，在 C++20/23 的新代码中，一般优先选择 `std::ranges::*` 算法；只有在维护旧代码或与旧接口兼容时，才继续使用传统的 `std::*` 算法。