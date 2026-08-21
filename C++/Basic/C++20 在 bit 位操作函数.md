C++20 在 <bit> 头文件中新增了一组位操作函数**，极大方便了底层位操作、竞争编程、加密、图形学等场景**。这些函数都是 constexpr 的，支持无符号整数类型（unsigned char、unsigned short、unsigned int、unsigned long、unsigned long long）。1. 旋转操作（Rotation）

cpp

```cpp
template<class T>
constexpr T rotl(T x, int s) noexcept;   // 左循环移位

template<class T>
constexpr T rotr(T x, int s) noexcept;   // 右循环移位
```

- 行为：
  - rotl(x, s)：把 x 的二进制位向左循环移动 s 位（超出部分回到右边）。
  - rotr(x, s)：向右循环移动。
  - s 可以是负数（rotl(x, -k) 等价于 rotr(x, k)）。
  - s 会被自动对位宽取模（s % N，其中 N = std::numeric_limits<T>::digits）。

示例：

cpp

```cpp
uint8_t x = 0b1001'0001;  // 0x91

std::rotl(x, 1)  => 0b0010'0011   // 0x23
std::rotr(x, 1)  => 0b1100'1000   // 0xC8
```

用途：哈希函数、加密算法（如 AES）、循环缓冲区等。2. 前导/后缀零/一计数（Count Leading/Trailing）

cpp

```cpp
template<class T>
constexpr int countl_zero(T x) noexcept;     // 前导0个数（从最高位开始）

template<class T>
constexpr int countl_one(T x) noexcept;      // 前导1个数

template<class T>
constexpr int countr_zero(T x) noexcept;     // 后缀0个数（从最低位开始）

template<class T>
constexpr int countr_one(T x) noexcept;      // 后缀1个数
```

示例：

cpp

```cpp
uint16_t val = 0b0000'1111'0000'0000;  // 0x0F00

std::countl_zero(val)   => 4
std::countl_one(val)    => 0
std::countr_zero(val)   => 8
std::countr_one(val)    => 0
```

用途：

- countl_zero：常用于计算最高有效位（MSB）位置 → 31 - countl_zero(x)（32位）。
- countr_zero：查找最低位1的位置（常用于位集合、棋盘算法）。
- 其他常用位操作函数

| 函数              | 功能说明                        | 典型用途                |
| ----------------- | ------------------------------- | ----------------------- |
| popcount(x)       | 计算1的个数（population count） | 汉明重量、位集合大小    |
| bit_ceil(x)       | 向上取整到下一个2的幂           | 内存对齐、哈希表扩容    |
| bit_floor(x)      | 向下取整到上一个2的幂           | 快速除法优化            |
| bit_width(x)      | 表示 x 所需的最小位数           | 动态位宽计算            |
| has_single_bit(x) | 判断 x 是否是2的精确幂          | 检查是否为 power of two |

4. 完整示例

cpp

```cpp
#include <bit>
#include <cstdint>
#include <iostream>

int main() {
    uint32_t x = 0b1010'0001'0000'0000'0000'0000'0000'0000u;

    std::cout << "popcount: " << std::popcount(x) << '\n';        // 3
    std::cout << "countl_zero: " << std::countl_zero(x) << '\n';  // 3
    std::cout << "bit_width: " << std::bit_width(x) << '\n';      // 29

    uint32_t y = 0b0000'0000'0000'0000'0000'0000'0001'0000u; // 16
    std::cout << "is power of 2? " 
              << std::has_single_bit(y) << '\n';               // true

    std::cout << "rotl(0x1234, 4) = 0x" 
              << std::hex << std::rotl(0x1234u, 4) << '\n';
}
```

5. 优势

- 可移植：取代了平台相关的 __builtin_clz、__builtin_popcount、rotl 等 intrinsic。
- constexpr：支持编译期计算。
- 类型安全：仅限无符号整数，避免符号位问题。

这些函数在竞争编程（如 Codeforces）、低级优化、加密等领域非常实用。需要某个函数的更深入实现原理或性能对比吗？