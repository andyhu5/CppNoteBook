`std::views::as_const` 是 **C++23** 新增的一个 **Range Adapter**，定义在 `<ranges>` 中，来源于提案 **P2278R4**。

它的作用非常简单：

> **将一个 Range 中的每个元素都视为 `const` 引用。**

也就是说，它不会复制数据，也不会修改容器，而是**禁止通过该视图修改元素**。

------

## 为什么需要 `views::as_const`？

假设有一个 `vector`：

```cpp
std::vector<int> v{1,2,3,4};
```

如果：

```cpp
for (auto& x : v)
{
    x *= 2;
}
```

这里：

```cpp
x
```

类型是：

```cpp
int&
```

可以修改元素。

有时我们希望：

> **只读遍历（read-only）**

以前只能写：

```cpp
for (const auto& x : v)
{
    ...
}
```

但是对于 Range Pipeline：

```cpp
v
| views::filter(...)
| views::transform(...)
```

已经没有地方写：

```cpp
const auto&
```

于是 C++23 增加：

```cpp
std::views::as_const
```

------

## 最简单的例子

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main()
{
    std::vector<int> v{1,2,3};

    auto view = v | std::views::as_const;

    for (auto&& x : view)
        std::cout << x << ' ';
}
```

输出：

```text
1 2 3
```

看起来没区别。

真正区别是：

```cpp
auto&& x
```

推导得到：

```cpp
const int&
```

而不是：

```cpp
int&
```

------

## 不能修改元素

例如：

```cpp
for (auto&& x : view)
{
    x = 100;
}
```

编译错误：

```text
assignment of read-only reference
```

因为：

```cpp
decltype(x)
```

已经变成：

```cpp
const int&
```

------

## 与普通 Range 的区别

没有：

```cpp
std::views::as_const
auto view = v;
```

得到：

```cpp
int&
```

可以：

```cpp
for (auto&& x : view)
    x++;
```

完全合法。

加入：

```cpp
std::views::as_const
```

以后：

```cpp
auto view = v
          | std::views::as_const;
```

所有元素都是：

```cpp
const int&
```

不能修改。

------

## 与 `std::as_const` 的区别

很多人容易混淆。

### `std::as_const`

C++17：

```cpp
std::as_const(obj)
```

作用对象：

> 一个对象

例如：

```cpp
std::vector<int> v;

auto& c = std::as_const(v);
```

得到：

```cpp
const std::vector<int>&
```

整个容器变成 const。

------

### `views::as_const`

作用对象：

> 一个 Range

例如：

```cpp
v
| std::views::as_const
```

得到：

```text
Range View
```

每个元素：

```cpp
const T&
```

更适合 Pipeline。

------

## 与 `transform` 对比

以前有人这样写：

```cpp
auto view =
    v
    | std::views::transform(
        [](const auto& x)
        {
            return x;
        });
```

目的只是：

> 不让修改元素。

但是：

- 多了 Lambda
- 多了 transform
- 不够直观

现在：

```cpp
auto view =
    v
    | std::views::as_const;
```

语义非常明确。

------

## 与 `filter` 配合

例如：

```cpp
auto even =
    v
    | std::views::filter(
        [](int x)
        {
            return x % 2 == 0;
        })
    | std::views::as_const;
```

此时：

```cpp
for (auto&& x : even)
```

得到：

```cpp
const int&
```

------

## 与 `transform` 配合

例如：

```cpp
auto view =
    words
    | std::views::as_const
    | std::views::transform(
        [](const std::string& s)
        {
            return s.size();
        });
```

整个 Pipeline 保持只读。

------

## 对底层容器有没有影响？

没有。

例如：

```cpp
std::vector<int> v{1,2,3};

auto cv = v | std::views::as_const;
```

之后：

```cpp
v[0] = 100;
```

仍然合法。

因为：

```text
as_const View
```

只是：

> **观察方式（View）**

不是：

```text
const vector
```

------

## 一个完整例子

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main()
{
    std::vector<int> v{1,2,3};

    auto view = v | std::views::as_const;

    // 编译错误
    /*
    for (auto&& x : view)
        x *= 2;
    */

    for (auto&& x : view)
        std::cout << x << ' ';

    std::cout << '\n';

    v[0] = 100;

    for (auto&& x : view)
        std::cout << x << ' ';
}
```

输出：

```text
1 2 3
100 2 3
```

说明：

- View 自己不能修改元素；
- 底层容器仍然可以修改；
- View 会实时反映底层数据变化。

------

## 实现原理

`views::as_const` 本质上等价于把元素引用类型从：

```cpp
T&
```

转换为：

```cpp
const T&
```

而不是复制元素。

因此：

- 零拷贝（Zero-copy）
- 零额外存储
- 惰性求值（Lazy Evaluation）
- 时间复杂度 O(1)

------

## 与其他适配器比较

| 适配器             | 作用                    |
| ------------------ | ----------------------- |
| `views::all`       | 转换成 View             |
| `views::filter`    | 过滤元素                |
| `views::transform` | 转换元素                |
| `views::take`      | 前 N 个元素             |
| `views::drop`      | 跳过前 N 个元素         |
| `views::reverse`   | 反向遍历                |
| `views::as_const`  | 将元素视为 `const` 引用 |

------

## 总结

| 特性              | `std::views::as_const` |
| ----------------- | ---------------------- |
| C++版本           | C++23                  |
| 头文件            | `<ranges>`             |
| 是否复制数据      | ❌                      |
| 是否修改底层容器  | ❌                      |
| 返回类型          | View                   |
| 元素类型          | `const T&`             |
| 是否支持 Pipeline | ✔                      |

**一句话概括：** `std::views::as_const` 是 C++23 新增的 Range 适配器，它不会复制数据，也不会使容器本身变为 `const`，而是将 **Range 中的每个元素统一视为 `const` 引用**，从而在 Range Pipeline 中提供一种简洁、安全的只读访问方式。