C++20 新增头文件 `<numbers>`，提供了一组**标准数学常量（Mathematical Constants）**。该提案来源于 **P0631R8 - Mathematical Constants for `<cmath>`**，最终进入 C++20。

在 C++20 之前，标准库没有提供数学常量，开发者通常使用：

```cpp
#define _USE_MATH_DEFINES
#include <cmath>

double pi = M_PI;
```

或者自己定义：

```cpp
constexpr double pi = 3.14159265358979323846;
```

这些方式都存在可移植性或精度问题，因此 C++20 引入了 `<numbers>`。

------

## 头文件

```cpp
#include <numbers>
```

------

## 所有数学常量

C++20 共定义了 10 个数学常量模板。

| 常量                       | 数学意义                |
| -------------------------- | ----------------------- |
| `std::numbers::e`          | 自然常数 e              |
| `std::numbers::log2e`      | log₂(e)                 |
| `std::numbers::log10e`     | log₁₀(e)                |
| `std::numbers::pi`         | π                       |
| `std::numbers::inv_pi`     | 1/π                     |
| `std::numbers::inv_sqrtpi` | 1/√π                    |
| `std::numbers::ln2`        | ln(2)                   |
| `std::numbers::ln10`       | ln(10)                  |
| `std::numbers::sqrt2`      | √2                      |
| `std::numbers::sqrt3`      | √3                      |
| `std::numbers::inv_sqrt3`  | 1/√3                    |
| `std::numbers::egamma`     | Euler–Mascheroni 常数 γ |
| `std::numbers::phi`        | 黄金分割比 φ            |

实际上共有 **13 个**常量（标准后来增加了完整集合），它们都具有一致的设计。

------

## 使用示例

```cpp
#include <iostream>
#include <numbers>

int main()
{
    std::cout << std::numbers::pi << '\n';
    std::cout << std::numbers::e << '\n';
    std::cout << std::numbers::sqrt2 << '\n';
}
```

输出（近似）：

```text
3.141592653589793
2.718281828459045
1.414213562373095
```

------

# 常量模板

真正定义的是变量模板（Variable Template）：

```cpp
namespace std::numbers
{
    template<class T>
    inline constexpr T pi_v = /*...*/;
}
```

然后提供一个默认的 `double` 版本：

```cpp
inline constexpr double pi = pi_v<double>;
```

因此：

```cpp
std::numbers::pi
```

实际上等价于：

```cpp
std::numbers::pi_v<double>
```

------

## 使用 float

```cpp
float x = std::numbers::pi_v<float>;
```

得到：

```
3.1415927f
```

------

## 使用 long double

```cpp
long double x = std::numbers::pi_v<long double>;
```

获得更高精度：

```
3.141592653589793238462643...
```

------

## 用户自定义浮点类型

只要满足要求，也可以实例化：

```cpp
MyFloat x = std::numbers::pi_v<MyFloat>;
```

例如：

- Boost.Multiprecision
- 任意高精度浮点类型

这是变量模板最大的优势。

------

# constexpr

所有常量都是：

```cpp
inline constexpr
```

因此可以：

```cpp
constexpr double radius = 10.0;

constexpr auto area =
    std::numbers::pi * radius * radius;
```

编译期即可计算。

------

# 类型安全

以前：

```cpp
constexpr double pi = 3.14159265358979323846;
```

如果：

```cpp
float x = pi;
```

需要转换。

现在：

```cpp
float x = std::numbers::pi_v<float>;
```

无需转换。

------

# 与 std::acos(-1) 对比

很多程序以前这样写：

```cpp
double pi = std::acos(-1);
```

缺点：

- 运行时调用
- 不是 constexpr（C++20 前）
- 可读性一般

现在：

```cpp
constexpr auto pi = std::numbers::pi;
```

更加直接。

------

# 常量精度

标准要求：

> 数值必须尽可能精确地表示对应类型所能表示的值。

例如：

```cpp
std::numbers::pi_v<float>
```

不是简单地把 double 转 float，

而是：

> **直接生成 float 最接近 π 的值。**

对于：

```cpp
pi_v<long double>
```

也是如此。

------

# 示例：计算圆面积

```cpp
#include <iostream>
#include <numbers>

double area(double r)
{
    return std::numbers::pi * r * r;
}

int main()
{
    std::cout << area(5.0);
}
```

输出：

```
78.5398
```

------

# 示例：模板算法

```cpp
#include <iostream>
#include <numbers>

template<typename T>
T circle_area(T r)
{
    return std::numbers::pi_v<T> * r * r;
}

int main()
{
    std::cout << circle_area(2.0f) << '\n';
    std::cout << circle_area(2.0) << '\n';
    std::cout << circle_area(2.0L) << '\n';
}
```

不同浮点类型都会使用对应精度的 π。

------

# 编译器支持

| 编译器 | 最低版本         |
| ------ | ---------------- |
| GCC    | 10               |
| Clang  | 11               |
| MSVC   | VS2019 16.8 左右 |

------

# 总结

`<numbers>` 是 C++20 对数学库的一项重要补充，它首次为标准库提供了统一、可移植且高精度的数学常量。其主要特点包括：

- 提供标准化的数学常量（π、e、√2、黄金分割比等）。
- 使用变量模板 `xxx_v<T>`，支持 `float`、`double`、`long double` 以及符合要求的用户自定义浮点类型。
- 所有常量均为 `inline constexpr`，可在编译期求值。
- 避免依赖非标准的 `M_PI` 等宏，提高代码的可移植性。
- 精度与目标类型匹配，无需依赖运行时函数（如 `std::acos(-1)`）来获取常量值。

对于现代 C++ 项目，推荐始终使用 `<numbers>` 中的数学常量，而不要再使用 `M_PI` 或自行定义常量。