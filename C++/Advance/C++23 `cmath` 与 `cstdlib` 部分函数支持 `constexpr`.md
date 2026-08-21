# C++23：`<cmath>` 与 `<cstdlib>` 部分函数支持 `constexpr`（P0533R9）



注意：GCC / MSVC 支持不同



```c
#include <cmath>
#include <cmath>
#include <array>


constexpr double r = std::sqrt(4.0);           // C++23 起合法，r == 2.0
constexpr double area = 3.14159 * std::pow(2.0, 2); // 编译期计算圆面积
//constexpr bool nan_check = std::isnan(0.0 / 0.0);    // 编译期判断 NaN

static_assert(std::floor(3.7) == 3.0);
static_assert(std::abs(-5) == 5);

// 编译期生成查找表
constexpr auto make_sine_table() {
    std::array<double, 360> table{};
    for (int i = 0; i < 360; ++i) {
        table[i] = std::sin(i * 3.14159265358979323846 / 180.0);
    }
    return table;
}
constexpr auto sine_table = make_sine_table(); // 完全在编译期生成

int main() {}
```



## 背景问题：数学函数长期无法用于编译期

`<cmath>`（C 数学库的 C++ 封装）和 `<cstdlib>` 中的一些函数，长期以来都是**普通的运行期函数**，无法用于 `constexpr` 上下文。这是因为：

1. 这些函数底层往往依赖**特定于硬件/操作系统的实现**（比如调用 CPU 的浮点指令、依赖 C 标准库的 `libm` 实现），编译期求值需要编译器自己有一套独立于运行时环境的算法实现；
2. 部分函数涉及**依赖实现定义的边界行为**（比如浮点误差、特殊值 `NaN`/`Inf` 的处理方式在不同平台可能有细微差异），标准委员会需要谨慎地划定"哪些函数可以保证在编译期和运行期给出一致、可预测的结果"。

这导致像下面这样很自然的写法，在 C++23 之前是不允许的：

```cpp
constexpr double x = std::sqrt(2.0);   // C++23 之前：编译错误，sqrt 不是 constexpr
constexpr double y = std::abs(-3.5);   // 同上
```

程序员如果想在编译期做一些数学计算（比如生成查找表、编译期常量的几何计算等），要么自己手写一套 `constexpr` 版本的数学函数（重复造轮子、容易和标准库版本行为不一致），要么放弃编译期计算。

## C++23 的解决方案：让大量数学函数支持 `constexpr`

C++23（P0533R9）给 `<cmath>` 和 `<cstdlib>` 中**大量数学函数**加上了 `constexpr`，覆盖范围包括：

### 1. 绝对值、取整、余数类

```cpp
std::abs, std::fabs        // 绝对值
std::floor, std::ceil      // 向下/向上取整
std::trunc, std::round     // 截断/四舍五入取整
std::fmod, std::remainder  // 浮点取余
```

### 2. 幂、指数、对数类

```cpp
std::sqrt, std::cbrt       // 平方根、立方根
std::pow                   // 幂运算
std::exp, std::exp2, std::expm1  // 指数函数
std::log, std::log2, std::log10, std::log1p  // 对数函数
```

### 3. 三角函数与双曲函数

```cpp
std::sin, std::cos, std::tan
std::asin, std::acos, std::atan, std::atan2
std::sinh, std::cosh, std::tanh
std::asinh, std::acosh, std::atanh
```

### 4. 浮点数分类与比较函数

```cpp
std::isnan, std::isinf, std::isfinite, std::isnormal
std::signbit
std::fpclassify
std::isgreater, std::isless, std::islessequal, std::isgreaterequal, std::islessgreater
std::isunordered
```

### 5. 其他实用函数

```cpp
std::lerp                  // C++20 引入的线性插值函数（本身已是 constexpr，此处一并提及）
std::fma                   // 融合乘加（fused multiply-add）
std::fmax, std::fmin       // 最大/最小值（正确处理 NaN）
std::hypot                 // 欧几里得距离（sqrt(x²+y²)，避免中间溢出）
std::copysign              // 拷贝符号位
```

### 6. `<cstdlib>` 中的部分函数

```cpp
std::abs (int/long/long long 版本)
std::div (整数除法，同时得到商和余数)
```

## 使用示例

```cpp
#include <cmath>

constexpr double r = std::sqrt(4.0);           // C++23 起合法，r == 2.0
constexpr double area = 3.14159 * std::pow(2.0, 2); // 编译期计算圆面积
constexpr bool nan_check = std::isnan(0.0 / 0.0);    // 编译期判断 NaN

static_assert(std::floor(3.7) == 3.0);
static_assert(std::abs(-5) == 5);

// 编译期生成查找表
constexpr auto make_sine_table() {
    std::array<double, 360> table{};
    for (int i = 0; i < 360; ++i) {
        table[i] = std::sin(i * 3.14159265358979323846 / 180.0);
    }
    return table;
}
constexpr auto sine_table = make_sine_table(); // 完全在编译期生成
```

## 关键限制：并非所有函数、所有情况都保证 `constexpr` 可行

### 限制 1：标准只要求"数学上有精确定义"的输入必须支持 `constexpr`

对于某些**数学上未定义或边界模糊**的输入（比如 `std::pow(-1.0, 0.5)` 涉及复数域、或者某些极端的溢出/下溢情况），标准给实现留有一定的自由度，具体行为可能因实现而异，编译器在这些边界情况下可能拒绝在 `constexpr` 上下文中求值（导致编译错误）而非静默给出不可靠的结果。

### 限制 2：结果允许有微小的实现差异

`constexpr` 求值路径和运行期求值路径，在理论上**可能**因为编译期使用的算法与运行期硬件指令/`libm` 实现不完全一致，而产生**极其微小的浮点精度差异**（通常在最后一两位有效数字）。标准并不强制要求"编译期求值结果与运行期求值结果逐位相同"，只要求都是"数学上合理精确"的结果。这是浮点计算固有的现实，不算是这次改动引入的新问题。

### 限制 3：浮点异常/错误处理相关的细节不参与 `constexpr`

比如设置 `errno`、触发浮点异常标志位（如除零标志）这类**具有副作用、依赖运行时环境状态**的行为，在 `constexpr` 求值中天然是不适用/被忽略的（`constexpr` 求值本身就在一个"纯粹"的抽象机器环境中进行，不涉及真实的硬件浮点异常状态机）。

## 典型应用场景

1. **编译期查找表生成**：三角函数表、对数表等，避免运行期重复计算，将结果直接烘焙进二进制文件。
2. **编译期几何/物理常量计算**：比如需要在编译期确定某个基于 `sqrt`/`pow` 的常量表达式，用作数组大小、模板非类型参数等。
3. **`static_assert` 中做数学验证**：在编译期就验证一些基于数学函数计算的不变量是否成立。
4. **`consteval` 函数中集成数学计算**：结合前面提到的 `consteval`/`if consteval`，可以写出"编译期一定用数学库函数计算，运行期换一条路径"的代码，而不必自己重新实现一遍数学函数。

## 小结

这项改动填补了 C++ 元编程生态里一个长期存在的空白：**数学计算是很多编译期场景（查找表、几何常量、数值验证）里绕不开的需求，但标准数学库函数长期被排除在 `constexpr` 之外**，逼迫开发者要么放弃编译期计算，要么自己手写重复、容易出错的 `constexpr` 版本实现。C++23 把 `<cmath>`/`<cstdlib>` 中绝大多数常用数学函数纳入 `constexpr` 支持范围，让"直接用标准数学函数做编译期计算"成为可能，是本轮讨论过的一系列"扩大 `constexpr` 适用范围"改动（`bitset`、`unique_ptr`、`to_chars`/`from_chars` 等）中，覆盖面最广、实用价值也最高的一项。