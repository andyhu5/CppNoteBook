**C++20 `std::midpoint` 和 `std::lerp` 完整介绍**

这两个函数都定义在头文件 **`<numeric>`** 中，是 C++20 为数值计算新增的实用工具函数。

---

### 1. `std::midpoint` —— 计算中点（安全版）

#### 函数原型

```cpp
template <class T>
constexpr T midpoint(T a, T b) noexcept;
```

- 支持整数类型（`int`、`long`、`unsigned` 等）和浮点类型（`float`、`double`、`long double`）。

#### 主要解决的问题

传统写法 ` (a + b) / 2 ` 在整数相加时**容易溢出**（尤其是大整数或无符号数）。

`std::midpoint` 使用更安全的算法，避免溢出。

#### 示例

```cpp
#include <numeric>
#include <iostream>

int main() {
    int a = INT_MAX - 10;
    int b = INT_MAX - 1;

    int mid1 = (a + b) / 2;           // 可能溢出！未定义行为
    int mid2 = std::midpoint(a, b);   // 安全计算

    std::cout << mid2 << '\n';
}
```

**优点**：
- 对于整数：使用 `(a / 2 + b / 2) + (a % 2 + b % 2) / 2` 等安全方式
- 对于浮点数：返回精确的中点
- `constexpr` 支持

---

### 2. `std::lerp` —— 线性插值（Linear Interpolation）

#### 函数原型

```cpp
template <class T>
constexpr T lerp(T a, T b, T t) noexcept;
```

**数学含义**：
```cpp
lerp(a, b, t) = a + t * (b - a)
```

其中 `t` 通常在 `[0.0, 1.0]` 范围内（但不强制）。

#### 示例

```cpp
#include <numeric>
#include <iostream>

int main() {
    double a = 10.0;
    double b = 50.0;

    std::cout << std::lerp(a, b, 0.0) << '\n';   // 10.0
    std::cout << std::lerp(a, b, 0.5) << '\n';   // 30.0
    std::cout << std::lerp(a, b, 1.0) << '\n';   // 50.0
    std::cout << std::lerp(a, b, 2.0) << '\n';   // 90.0 （支持外插）
}
```

**应用场景**：
- 动画过渡（平滑移动、颜色渐变）
- 游戏开发（插值位置、角度）
- 图形学、音频处理
- 科学计算

---

### 3. 两者对比总结

| 函数       | 功能         | 主要优势           | 典型用途                   |
| ---------- | ------------ | ------------------ | -------------------------- |
| `midpoint` | 计算两点中点 | 防止整数溢出       | 数值算法、二分查找、树结构 |
| `lerp`     | 线性插值     | 语义清晰、支持外插 | 动画、图形、平滑过渡       |

---

### 4. 注意事项

- 两者都是 **`constexpr`**，可以在编译期使用。
- `midpoint` 对整数特别有用，对浮点数效果与 `(a+b)/2` 类似但更精确。
- `lerp` 在 `t` 超出 `[0,1]` 时会进行**外插**（extrapolation），这是符合预期的。
- 浮点数计算仍受浮点精度限制（例如 `lerp(0.0, 1.0, 0.1)` 可能不是精确的 0.1）。

---

**推荐用法**：

```cpp
// 二分查找中点（强烈推荐）
auto mid = std::midpoint(low, high);

// 动画插值
position = std::lerp(start_pos, end_pos, progress);
color = std::lerp(start_color, end_color, t);
```

---

需要我给出更多实际例子（如在二分查找、颜色渐变、游戏开发中的应用）吗？