C++20 对数值算法（Numeric Algorithms）的增强解释C++20 在头文件 <numeric> 中新增和改进了多个数值相关算法，主要目的是提供更安全、更精确、更方便的常用数学运算，尤其适合数值计算、图形学、游戏开发等领域。

1. 新增算法（C++20）(1) std::midpoint —— 计算中点（最重要！）

cpp

```cpp
template<class T>
constexpr T midpoint(T a, T b) noexcept;
```

- 功能：计算两个数的算术平均值（中点），但避免溢出。
- 传统写法 (a + b) / 2 在整数或大数值时容易溢出，std::midpoint 使用更安全的算法。

支持类型：

- 整数类型（int、long、unsigned 等）
- 浮点类型（float、double）
- 指针类型（计算两个指针的中间地址）

示例：

cpp

```cpp
#include <numeric>
#include <iostream>

int main() {
    int a = INT_MAX - 1;
    int b = INT_MAX;

    std::cout << (a + b) / 2 << '\n';           // 未定义行为！可能溢出
    std::cout << std::midpoint(a, b) << '\n';   // 正确：INT_MAX - 1
}
```

(2) std::lerp —— 线性插值（Linear Interpolation）

cpp

```cpp
constexpr float lerp(float a, float b, float t) noexcept;
constexpr double lerp(double a, double b, double t) noexcept;
```

- 功能：计算 a + t * (b - a)，即在 a 和 b 之间按比例 t 进行线性插值。
- t 通常在 [0, 1] 范围内，但不强制。

用途：动画、颜色过渡、图形学、游戏平滑移动等。示例：

cpp

```cpp
float start = 10.0f;
float end = 50.0f;
float t = 0.75f;                    // 75% 位置

float value = std::lerp(start, end, t);   // 40.0
```

------

2. 其他数值算法改进（C++20）<numeric> 头文件在 C++20 中还对以下算法进行了增强（主要是 constexpr 支持）：

- std::gcd（最大公约数）
- std::lcm（最小公倍数）
- std::reduce
- std::inclusive_scan、std::exclusive_scan
- std::transform_reduce
- std::transform_inclusive_scan、std::transform_exclusive_scan

这些算法在 C++20 中全部支持 constexpr，可以在编译期使用。

------

3. 完整示例

cpp

```cpp
#include <numeric>
#include <iostream>
#include <vector>

int main() {
    // 1. midpoint
    constexpr int m1 = std::midpoint(1, 10);           // 5
    constexpr double m2 = std::midpoint(1.0, 10.0);    // 5.5

    // 2. lerp
    constexpr double pos = std::lerp(0.0, 100.0, 0.3); // 30.0

    // 3. gcd / lcm
    constexpr auto g = std::gcd(48, 18);   // 6
    constexpr auto l = std::lcm(12, 18);   // 36

    std::cout << "midpoint: " << m1 << '\n';
    std::cout << "lerp: " << pos << '\n';
}
```

------

总结：C++20 数值算法亮点

| 算法       | 主要作用              | 关键优势             |
| ---------- | --------------------- | -------------------- |
| midpoint   | 计算中点              | 防止整数溢出         |
| lerp       | 线性插值              | 图形/动画常用        |
| gcd / lcm  | 最大公约数/最小公倍数 | 编译期可用           |
| *scan 系列 | 前缀和类运算          | 支持并行 + constexpr |

这些函数让 C++ 在数值计算方面更现代化、更安全，减少了手动实现容易出错的代码。

------

想深入了解哪个部分？
例如：midpoint 的实现原理、lerp 在游戏开发中的应用、还是 constexpr 数值计算示例？随时告诉我！