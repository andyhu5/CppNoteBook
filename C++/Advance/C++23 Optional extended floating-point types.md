## C++23 Optional extended floating-point types

C++23 新增了 `<stdfloat>` 头文件，引入一组**标准名称**的浮点类型：

```cpp
std::float16_t
std::float32_t
std::float64_t
std::float128_t
std::bfloat16_t
```

对应提案：**P1467R9 - Extended floating-point types and standard names**。

它们的目的不是增加新的浮点格式，而是**为已有的 IEEE-754 格式提供统一的 C++ 类型名称**。

------

## 为什么需要它们？

以前只有

```cpp
float
double
long double
```

但它们的精度依赖平台：

| 类型        | Windows/MSVC  | GCC x86-64    | ARM               |
| ----------- | ------------- | ------------- | ----------------- |
| float       | IEEE binary32 | IEEE binary32 | IEEE binary32     |
| double      | IEEE binary64 | IEEE binary64 | IEEE binary64     |
| long double | 64-bit        | 80-bit        | 128-bit 或 64-bit |

例如：

```cpp
sizeof(long double)
```

Windows：

```text
8
```

Linux x86：

```text
16      // 实际有效位80-bit
```

因此不能写出真正可移植的高精度代码。

------

## C++23 提供固定格式名称

如果实现支持，就提供：

| 类型              | IEEE格式  | 位数 |
| ----------------- | --------- | ---- |
| `std::float16_t`  | binary16  | 16   |
| `std::float32_t`  | binary32  | 32   |
| `std::float64_t`  | binary64  | 64   |
| `std::float128_t` | binary128 | 128  |
| `std::bfloat16_t` | bfloat16  | 16   |

例如：

```cpp
#include <stdfloat>

std::float32_t x = 1.5f;
std::float64_t y = 3.14;
```

------

## 对应关系

通常：

```text
float          -> float32
double         -> float64
long double    -> 平台决定
```

所以：

```cpp
std::float32_t
```

并**不一定**

```cpp
== float
```

虽然很多平台实际上就是。

------

## 为什么叫 Optional（可选）？

标准没有要求所有平台必须支持。

例如：

```cpp
#ifdef __STDCPP_FLOAT128_T__
using fp = std::float128_t;
#endif
```

实现只有真正支持 binary128 时才定义：

```cpp
std::float128_t
```

否则根本不存在。

同理：

```text
std::float16_t
std::bfloat16_t
```

也是如此。

------

## float16

IEEE binary16：

```
1 sign
5 exponent
10 fraction
```

共16位。

大约：

```
有效数字：3~4位十进制
```

范围：

```
≈ 6e−8
~
65504
```

主要用途：

- GPU
- 图像处理
- AI推理
- 神经网络

例如：

```cpp
std::float16_t weight;
```

比 float 节省一半内存。

------

## bfloat16

Google 提出的 Brain Floating Point。

结构：

```
1 sign
8 exponent
7 fraction
```

与 float16 最大区别：

| 类型     | exponent | mantissa |
| -------- | -------- | -------- |
| float16  | 5        | 10       |
| bfloat16 | 8        | 7        |

因此：

```
float16
```

精度高一点。

```
bfloat16
```

动态范围几乎与 float 一样。

------

例如：

```
float32

31........23........0
S EEEEEEEE MMMMMMMMMMMMMMMMMMMMMMM
```

bfloat16：

```
15......7....0
S EEEEEEEE MMMMMMM
```

直接截掉 float 的低16位即可。

因此 AI 芯片特别喜欢。

------

## float128

IEEE binary128：

```
1 sign
15 exponent
112 fraction
```

约：

```
34 位十进制有效数字
```

相比：

| 类型     | 十进制有效数字 |
| -------- | -------------- |
| float    | 7              |
| double   | 15             |
| float128 | 34             |

适用于：

- 科学计算
- 天体计算
- 数值分析
- 高精度积分

------

## 如何判断是否支持？

使用宏：

```cpp
#ifdef __STDCPP_FLOAT16_T__
#endif

#ifdef __STDCPP_FLOAT32_T__
#endif

#ifdef __STDCPP_FLOAT64_T__
#endif

#ifdef __STDCPP_FLOAT128_T__
#endif

#ifdef __STDCPP_BFLOAT16_T__
#endif
```

例如：

```cpp
#include <stdfloat>
#include <iostream>

int main()
{
#ifdef __STDCPP_FLOAT128_T__
    std::cout << "float128 supported\n";
#else
    std::cout << "float128 not supported\n";
#endif
}
```

------

## 与 `<cstdint>` 的关系

两者设计理念完全一致。

```cpp
std::uint32_t
```

保证：

```
32-bit 整数
```

对应：

```cpp
std::float32_t
```

保证：

```
IEEE binary32 浮点数
```

因此：

```
<cstdint>   -> 固定宽度整数
<stdfloat>  -> 固定格式浮点数
```

------

## 编译器支持

| 编译器    | 支持情况                               |
| --------- | -------------------------------------- |
| GCC 13+   | 较完整（依赖硬件）                     |
| Clang 17+ | 较完整                                 |
| MSVC      | 目前支持有限，尤其 `float128_t` 不支持 |

很多 ARM 平台支持：

```
float16
```

很多 AI/GPU 平台支持：

```
bfloat16
```

而 x86/Linux 通常可通过 `libquadmath` 或硬件/软件实现支持 `float128`。

------

## 总结

| 类型              | IEEE格式  | 主要用途                        |
| ----------------- | --------- | ------------------------------- |
| `std::float16_t`  | binary16  | GPU、移动设备、AI 推理          |
| `std::float32_t`  | binary32  | 对应单精度，通常等价于 `float`  |
| `std::float64_t`  | binary64  | 对应双精度，通常等价于 `double` |
| `std::float128_t` | binary128 | 高精度科学计算                  |
| `std::bfloat16_t` | bfloat16  | AI 训练、深度学习加速           |

**核心价值**：C++23 首次为常见 IEEE-754 浮点格式提供了统一、可移植的类型名称，使泛型库和跨平台数值计算无需依赖 `float`、`double`、`long double` 的平台实现细节。