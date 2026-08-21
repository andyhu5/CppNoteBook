## C++23: Literal suffix `Z` / `z` for signed `std::size_t` literals

该特性来自 **P0330R8**，为整数字面量新增后缀 **`z`** 和 **`Z`**，用于生成与 `std::size_t` 对应的**有符号整数类型**（即 `std::make_signed_t<std::size_t>`）。

### 为什么需要？

`std::size_t` 是无符号类型：

```cpp
std::size_t n = 10;
```

很多时候需要与它对应的**有符号类型**：

```cpp
std::ptrdiff_t
```

以前只能这样写：

```cpp
auto x = std::ssize(v);      // C++20
std::ptrdiff_t y = 10;
```

或者：

```cpp
using ssize_t = std::make_signed_t<std::size_t>;
ssize_t x = 10;
```

写法比较繁琐。

------

## C++23 新写法

```cpp
auto x = 10z;
```

这里：

```cpp
10z
```

类型就是：

```cpp
std::make_signed_t<std::size_t>
```

大小与 `size_t` 相同，但为**有符号整数**。

例如：

```cpp
#include <type_traits>

static_assert(
    std::is_same_v<
        decltype(0z),
        std::make_signed_t<std::size_t>
    >);
```

------

## `z` 和 `Z`

大小写都可以：

```cpp
10z
10Z
```

完全等价。

------

## 与其他后缀组合

可以和 `u/U` 组合，表示 **`size_t` 本身**。

```cpp
10uz
10Zu
10uZ
```

类型都是：

```cpp
std::size_t
```

例如：

```cpp
static_assert(
    std::is_same_v<
        decltype(0uz),
        std::size_t
    >);
```

------

## 类型对应关系

假设 64 位平台：

| 字面量 | 类型                              |
| ------ | --------------------------------- |
| `10`   | `int`                             |
| `10L`  | `long`                            |
| `10LL` | `long long`                       |
| `10u`  | `unsigned int`                    |
| `10uz` | `std::size_t`                     |
| `10z`  | `std::make_signed_t<std::size_t>` |

通常：

```text
64-bit Linux
size_t            -> unsigned long
10z               -> long

64-bit Windows
size_t            -> unsigned long long
10z               -> long long
```

因此 **`10z` 不是固定的 `long` 或 `long long`**，而是取决于平台的 `size_t`。

------

## 使用场景

### 1. 循环计数

以前：

```cpp
for (std::ptrdiff_t i = 0; i < std::ssize(v); ++i)
```

现在：

```cpp
for (auto i = 0z; i < std::ssize(v); ++i)
```

无需写类型。

------

### 2. 避免有符号/无符号混用

以前：

```cpp
std::vector<int> v;

if (v.size() > 10)
```

这里：

```cpp
10
```

是 `int`，会发生整数提升。

现在：

```cpp
if (v.size() > 10uz)
```

两边都是 `size_t`。

如果比较 `std::ssize(v)`：

```cpp
if (std::ssize(v) > 10z)
```

两边都是对应的有符号类型。

------

### 3. 泛型代码

```cpp
template<typename C>
void f(const C& c)
{
    for (auto i = 0z; i < std::ssize(c); ++i)
    {
    }
}
```

不需要知道平台上 `ptrdiff_t` 是 `long` 还是 `long long`。

------

## 推荐使用

当代码涉及：

- `std::size_t`
- `std::ssize()`
- 容器索引
- 泛型库

推荐使用：

```cpp
0uz      // std::size_t
0z       // signed size_t
```

比：

```cpp
(size_t)0
std::ptrdiff_t{0}
```

更简洁、更可移植。

------

## 总结

| 后缀                      | 类型                              |
| ------------------------- | --------------------------------- |
| `z` / `Z`                 | `std::make_signed_t<std::size_t>` |
| `uz` / `uZ` / `Zu` / `UZ` | `std::size_t`                     |

**核心价值**：C++23 为 `size_t` 及其对应的有符号类型提供了标准字面量后缀，使泛型代码、容器索引和 `std::ssize()` 的使用更加自然，同时减少有符号/无符号类型混用带来的警告和转换问题。