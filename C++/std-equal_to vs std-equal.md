

- **`std::equal_to`**：***函数对象***（Function Object/Functor），用于比较两个值是否相等。
- **`std::equal`**：***算法（Algorithm）***，用于比较两个序列是否相等。

二者的关系可以理解为：

> `std::equal` 算法默认内部就是使用 `std::equal_to<>` 来逐个比较元素。

------

## 1. `std::equal_to`

头文件：

```cpp
#include <functional>
```

定义（简化）：

```cpp
template<class T = void>
struct equal_to
{
    constexpr bool operator()(const T& lhs,
                              const T& rhs) const;
};
```

它就是一个可调用对象：

```cpp
std::equal_to<> eq;

std::cout << eq(1, 1); // true
std::cout << eq(1, 2); // false
```

等价于：

```cpp
[](auto&& a, auto&& b)
{
    return a == b;
}
```

因此你的代码：

```cpp
auto r = v2 | std::views::chunk_by(std::equal_to{});
```

实际上等价于：

```cpp
auto r = v2 | std::views::chunk_by(
    [](int a, int b)
    {
        return a == b;
    });
```

所以会得到：

```
[1 1 1]
[2 2]
[3]
[4 4]
```

------

## 2. `std::equal`

头文件：

```cpp
#include <algorithm>
```

它是一个算法，用来比较两个序列：

```cpp
std::vector<int> a{1,2,3};
std::vector<int> b{1,2,3};

bool ok = std::equal(
    a.begin(), a.end(),
    b.begin()
);
```

结果：

```text
true
```

内部逻辑类似：

```cpp
while (first1 != last1)
{
    if (!(*first1 == *first2))
        return false;

    ++first1;
    ++first2;
}

return true;
```

------

## 3. `std::equal` 也可以指定比较器

例如：

```cpp
std::equal(
    a.begin(), a.end(),
    b.begin(),
    std::equal_to<>{}
);
```

这里：

```
std::equal   ← 算法
std::equal_to ← 比较器
```

关系就像：

```text
std::sort
        │
        ▼
std::less<>
```

或者：

```text
std::ranges::sort
        │
        ▼
std::less<>
```

------

## 4. 为什么 `chunk_by` 使用 `equal_to`？

因为 `chunk_by` 需要的是一个**二元谓词（Binary Predicate）**。

对于相邻两个元素：

```cpp
pred(current, next)
```

如果返回：

```cpp
true
```

继续放在当前组。

否则：

```cpp
false
```

开始新的组。

因此需要：

```cpp
bool(int, int)
```

这样的可调用对象。

`std::equal_to` 正好满足：

```cpp
std::equal_to<>{}(a, b)
```

返回：

```cpp
a == b
```

而 `std::equal` 是：

```cpp
std::equal(first1, last1, first2)
```

它比较的是**两个序列**，根本不是同一种接口。

因此：

```cpp
std::views::chunk_by(std::equal);   // ❌ 错误
```

不能编译。

------

## 5. 为什么推荐 `std::equal_to{}`？

在现代 C++（C++14 以后），更推荐：

```cpp
std::equal_to{}
```

而不是：

```cpp
std::equal_to<int>{}
```

因为：

```cpp
std::equal_to<>
```

是**透明比较器（Transparent Comparator）**，能够自动推导参数类型。

例如：

```cpp
std::equal_to<> eq;

eq(10, 10L);          // OK
eq("abc", std::string("abc")); // OK（若支持对应 ==）
```

这比固定类型更加灵活。

------

## 对比总结

| 项目              | `std::equal_to`     | `std::equal`      |
| ----------------- | ------------------- | ----------------- |
| 头文件            | `<functional>`      | `<algorithm>`     |
| 类型              | 函数对象（Functor） | 算法（Algorithm） |
| 参数              | 两个值              | 两个序列          |
| 返回值            | `bool`              | `bool`            |
| 用途              | 比较两个对象        | 比较两个范围      |
| `chunk_by` 可用   | ✔                   | ❌                 |
| `sort` 比较器可用 | ✔                   | ❌                 |

**一句话概括：** `std::equal_to` 是一个"如何比较两个元素"的比较器，而 `std::equal` 是一个"如何比较两个序列"的算法。`std::views::chunk_by` 需要的是前者，因此可以写 `std::views::chunk_by(std::equal_to{})`，但不能写 `std::views::chunk_by(std::equal)`。