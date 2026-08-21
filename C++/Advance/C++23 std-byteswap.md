`std::byteswap` 是 **C++23** 在 `<bit>` 头文件中新增的一个函数，用于**交换整数类型的字节序（byte order）**，对应提案 **P1272R4**。

它可以理解为标准库版的：

- GCC / Clang：`__builtin_bswap16/32/64`
- MSVC：`_byteswap_ushort/_ulong/_uint64`

目的是提供一个**可移植、constexpr、标准化**的字节交换接口。

------

## 为什么需要 `std::byteswap`？

很多程序需要进行大小端（Endianness）转换，例如：

- 网络协议（Network Byte Order）
- 二进制文件
- 数据库
- 图像格式
- CPU 指令
- 嵌入式开发

以前通常写：

```cpp
uint32_t x = 0x12345678;

x =
    ((x & 0x000000FF) << 24) |
    ((x & 0x0000FF00) << 8)  |
    ((x & 0x00FF0000) >> 8)  |
    ((x & 0xFF000000) >> 24);
```

或者：

```cpp
__builtin_bswap32(x);
```

或者：

```cpp
_byteswap_ulong(x);
```

这些都具有平台依赖性。

C++23 统一为：

```cpp
std::byteswap(x);
```

------

## 函数原型

定义在：

```cpp
#include <bit>
```

原型大致为：

```cpp
template<class T>
constexpr T byteswap(T value) noexcept;
```

要求：

```text
T 必须是整数类型（integral type）
```

包括：

- `short`
- `int`
- `long`
- `long long`
- `std::uint32_t`
- `std::int64_t`

等等。

------

## 一个例子

```cpp
#include <bit>
#include <cstdint>
#include <iostream>

int main()
{
    std::uint32_t x = 0x12345678;

    auto y = std::byteswap(x);

    std::cout << std::hex << y;
}
```

输出：

```text
78563412
```

即：

原来

```text
12 34 56 78
```

变成

```text
78 56 34 12
```

------

## 不同整数类型

### uint16_t

```cpp
std::uint16_t x = 0x1234;

auto y = std::byteswap(x);
```

结果：

```text
12 34
↓

34 12
```

得到：

```text
0x3412
```

------

### uint32_t

```text
12 34 56 78

↓

78 56 34 12
```

得到：

```text
0x78563412
```

------

### uint64_t

```text
11 22 33 44 55 66 77 88

↓

88 77 66 55 44 33 22 11
```

得到：

```text
0x8877665544332211
```

------

## 它不会反转位（bit）

很多人第一次看到容易误解。

例如：

```text
0x12
```

二进制：

```text
00010010
byteswap
```

**不会**得到

```text
01001000
```

因为：

```
byteswap
```

交换的是：

```text
Byte
```

不是：

```text
Bit
```

例如：

```text
0x1234

Byte:

12
34

↓

34
12
```

每个 Byte 内部：

```text
00010010
```

保持不变。

------

## constexpr

最大的优点之一：

它是

```cpp
constexpr
```

例如：

```cpp
constexpr std::uint32_t x =
    std::byteswap(0x12345678u);

static_assert(x == 0x78563412);
```

编译期间即可完成。

以前很多 builtin 做不到这一点（或者不是标准保证）。

------

## 与 `std::endian` 配合使用

C++20 引入了：

```cpp
#include <bit>

std::endian
```

例如：

```cpp
if constexpr (std::endian::native == std::endian::little)
{
    value = std::byteswap(value);
}
```

即可实现：

```text
Little Endian
↓

Big Endian
```

这是现代 C++ 推荐写法。

------

## 网络字节序转换

例如：

```cpp
#include <bit>
#include <cstdint>

std::uint32_t host_to_network(std::uint32_t x)
{
    if constexpr (std::endian::native ==
                  std::endian::little)
    {
        return std::byteswap(x);
    }

    return x;
}
```

无需：

```cpp
htonl()
```

虽然在 POSIX 上：

```cpp
htonl()
```

仍然可以使用。

------

## 编译器如何实现？

通常不会真的逐字节交换。

例如：

```cpp
std::byteswap(x);
```

GCC

生成：

```text
bswap
```

CPU 指令。

MSVC

生成：

```text
bswap
```

Clang

同样。

因此几乎没有性能损失。

例如 x86：

```asm
bswap eax
```

就是一条指令。

------

## 限制

### 不能用于浮点数

错误：

```cpp
double x = 3.14;

std::byteswap(x);
```

因为：

```text
double
```

不是整数类型。

如果需要：

```text
double
```

交换字节，应先：

```cpp
auto bits = std::bit_cast<std::uint64_t>(x);

bits = std::byteswap(bits);

auto y = std::bit_cast<double>(bits);
```

------

### 不能用于结构体

例如：

```cpp
struct S
{
    int a;
    short b;
};

std::byteswap(S{});
```

也是非法。

因为：

```text
byteswap
```

只支持整数。

------

## 与 `std::bit_cast` 的关系

两者都位于：

```cpp
#include <bit>
```

但是作用不同：

| 功能         | `std::bit_cast` | `std::byteswap` |
| ------------ | --------------- | --------------- |
| 类型转换     | ✔               | ❌               |
| 修改字节顺序 | ❌               | ✔               |
| 改变对象表示 | ❌               | ✔（交换字节）   |
| `constexpr`  | ✔               | ✔               |

它们经常一起使用，例如处理浮点数、GUID 或网络协议中的复杂数据。

------

## 总结

| 特性                      | `std::byteswap`            |
| ------------------------- | -------------------------- |
| C++版本                   | C++23                      |
| 头文件                    | `<bit>`                    |
| 是否 `constexpr`          | ✔                          |
| 是否 `noexcept`           | ✔                          |
| 支持类型                  | 整数类型（integral）       |
| 作用                      | 交换字节顺序（Endianness） |
| 是否交换 bit              | ❌                          |
| 是否调用 CPU `bswap` 指令 | 通常会                     |

**一句话概括：** `std::byteswap` 是 C++23 提供的标准化字节序交换函数，用于将整数值的字节顺序反转。它取代了平台专有的 `__builtin_bswap*`、`_byteswap_*` 等接口，支持 `constexpr`，通常还能映射到处理器的单条 `bswap` 指令，是进行大小端转换和二进制数据处理的标准工具。