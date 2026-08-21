**`std::tuple` is compatible with other tuple-like objects** 是 **C++23** 的一项标准库改进，来源于提案 **P2165R4**。

它的目标是：

> **让 `std::tuple` 能够更自然地与其他“tuple-like（元组式）”类型互操作。**

简单来说，以前很多只能在 `std::tuple` 之间进行的构造、赋值和转换，C++23 扩展到了所有符合 **tuple-like** 概念的类型。

------

## 什么是 tuple-like？

一个类型如果满足以下条件，就可以认为是 **tuple-like**：

- 可以使用 `std::tuple_size`
- 可以使用 `std::tuple_element`
- 可以使用 `std::get`

例如：

```cpp
std::pair
std::array
std::tuple
```

都是 tuple-like。

例如：

```cpp
std::pair<int, double>
```

具有：

```cpp
std::tuple_size_v<std::pair<int,double>> == 2

std::get<0>(pair)
std::get<1>(pair)
```

因此它就是 tuple-like。

------

## C++20 的问题

假设：

```cpp
std::pair<int, double> p{1, 3.14};
```

我们希望：

```cpp
std::tuple<int, double> t = p;
```

很多情况下是不允许的，或者需要借助：

```cpp
std::make_tuple(p.first, p.second);
```

或者：

```cpp
auto t = std::tuple{
    p.first,
    p.second
};
```

标准库不同组件之间的互操作性较差。

------

## C++23 改进

现在可以直接：

```cpp
#include <tuple>
#include <utility>

std::pair<int, double> p{1, 3.14};

std::tuple<int, double> t = p;
```

或者：

```cpp
std::tuple<int, double> t(p);
```

都可以。

------

## `std::array`

例如：

```cpp
std::array<int,3> a{1,2,3};
```

现在：

```cpp
std::tuple<int,int,int> t(a);
```

即可。

以前通常不支持。

------

## 反向转换

例如：

```cpp
std::tuple<int,double> t{1,3.14};

std::pair<int,double> p(t);
```

C++23 同样支持。

------

## 赋值

例如：

```cpp
std::pair<int,int> p{1,2};

std::tuple<int,int> t;

t = p;
```

现在合法。

反过来也可以：

```cpp
p = t;
```

------

## 更复杂的例子

```cpp
std::array<int,2> a{10,20};

std::pair<long,long> p(a);
```

实际上等价于：

```cpp
std::pair<long,long>{
    a[0],
    a[1]
};
```

每个元素分别转换。

------

## 为什么称为 tuple-like？

因为标准没有要求必须是：

```cpp
std::tuple
```

任何类型只要满足：

```cpp
tuple_size
tuple_element
get
```

都可以。

例如用户自定义类型：

```cpp
struct Point
{
    int x;
    int y;
};
```

如果提供：

```cpp
std::tuple_size<Point>
std::tuple_element<0,Point>
std::tuple_element<1,Point>

get<0>(Point)
get<1>(Point)
```

那么：

```cpp
Point pt{1,2};

std::tuple<int,int> t(pt);
```

也是合法的。

------

## 为什么要这样修改？

主要原因是：

### 1. 提高一致性

以前：

```cpp
tuple

↓

tuple
```

可以。

但是：

```cpp
pair

↓

tuple
```

很多情况却不行。

虽然：

```cpp
pair
```

本身就是 tuple-like。

标准行为不一致。

------

### 2. 配合 Ranges

Ranges 库大量使用：

```cpp
pair
tuple
subrange
zip_view
```

它们都属于：

```text
tuple-like
```

统一接口之后：

不同组件之间可以直接互相转换。

------

### 3. 支持结构化绑定生态

例如：

```cpp
auto [x,y] = obj;
```

要求：

```text
tuple-like
```

因此：

tuple-like 已经成为现代 C++ 的一个重要概念。

------

## 哪些类型受益？

标准库包括：

```cpp
std::tuple
std::pair
std::array
std::ranges::subrange
```

以及：

所有满足 tuple-like 要求的用户自定义类型。

------

## 需要满足什么条件？

例如：

```cpp
std::tuple<int,double>
```

构造：

```cpp
std::pair<long,float>
```

要求：

1. 元素数量一致

```text
2 == 2
```

1. 每个元素都可转换：

```text
long

↓

int

float

↓

double
```

否则：

编译失败。

------

## 一个完整例子

```cpp
#include <array>
#include <iostream>
#include <tuple>

int main()
{
    std::array<int,3> a{1,2,3};

    std::tuple<int,int,int> t(a);

    auto [x,y,z] = t;

    std::cout << x << ' '
              << y << ' '
              << z << '\n';
}
```

输出：

```text
1 2 3
```

整个过程无需手动拆解数组元素。

------

## 与 `std::apply` 的关系

C++17 引入了：

```cpp
std::apply(f, tuple_like);
```

实际上，`std::apply` 早已支持很多 tuple-like 类型，例如：

```cpp
std::pair
std::array
```

但：

```cpp
std::tuple
```

自身的构造函数和赋值运算符却没有完全支持 tuple-like。

C++23 将这一行为统一：

- `std::apply`：支持 tuple-like。
- `std::tuple`：构造、赋值、转换也支持 tuple-like。

整个标准库的设计更加一致。

------

## 总结

| 功能                       | C++20          | C++23 |
| -------------------------- | -------------- | ----- |
| `tuple ← tuple`            | ✔              | ✔     |
| `tuple ← pair`             | 部分支持或受限 | ✔     |
| `tuple ← array`            | 部分支持或受限 | ✔     |
| `pair ← tuple`             | 部分支持或受限 | ✔     |
| 用户自定义 tuple-like 转换 | 很有限         | ✔     |
| tuple-like 生态一致性      | 较弱           | 更强  |

**一句话概括：** C++23 将 `std::tuple` 扩展为能够与所有 **tuple-like** 类型（如 `std::pair`、`std::array`、`std::ranges::subrange` 以及符合 `tuple_size`/`tuple_element`/`get` 协议的自定义类型）进行统一的构造、赋值和转换。这使得标准库中所有遵循 tuple-like 协议的类型能够更加自然地互操作，也进一步完善了现代 C++ 围绕 tuple-like 概念构建的生态。