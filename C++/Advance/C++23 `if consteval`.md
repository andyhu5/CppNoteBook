#### C++23：`if consteval`（P1938R3）

## 背景问题

C++20 引入了 `std::is_constant_evaluated()`，用来在函数内部判断当前是否处于**常量求值上下文**（即编译期求值 vs 运行期求值），从而让同一个函数根据求值时机选择不同的实现路径：

```cpp
constexpr double power(double base, int exp) {
    if (std::is_constant_evaluated()) {
        // 编译期：用一个简单、可靠、允许在 constexpr 中运行的朴素算法
        double result = 1.0;
        for (int i = 0; i < exp; ++i) result *= base;
        return result;
    } else {
        // 运行期：调用高性能的、可能依赖硬件指令的 std::pow
        return std::pow(base, exp);
    }
}
```

这个设计存在几个容易踩坑的问题：

### 问题 1：`if` 分支必须两边类型/语义都合法，容易被误用为普通运行期分支

`std::is_constant_evaluated()` 本质上就是一个返回 `bool` 的普通函数，`if` 只是普通的运行期 `if` 语句（虽然编译器会做一些特殊处理保证它在常量求值时能正确工作）。这导致一个常见误用：

```cpp
constexpr int f(int n) {
    if (std::is_constant_evaluated()) {
        return n;
    }
    return n; // 忘了写 else，这里的代码在两种情况下都会被"看到"
}
```

更隐蔽的问题是：如果你写成 `if (!std::is_constant_evaluated())`，某些写法在优化或语义上可能不符合预期，因为编译器仍然把它当作普通的运行期条件表达式来处理，只是**保证在常量求值时这个表达式的结果是可预测的**——细节上容易出错。

### 问题 2：无法在 `if` 条件里直接否定并简洁表达

```cpp
if (!std::is_constant_evaluated()) {
    // 运行期分支
}
```

这种写法虽然合法，但语义上略显绕。

## C++23 的解决方案：`if consteval`

C++23 引入了新的语法 `if consteval { ... } else { ... }`，直接作为**语言关键字级别**的分支结构，专门用来区分编译期 / 运行期路径：

```cpp
constexpr double power(double base, int exp) {
    if consteval {
        // 编译期路径
        double result = 1.0;
        for (int i = 0; i < exp; ++i) result *= base;
        return result;
    } else {
        // 运行期路径
        return std::pow(base, exp);
    }
}
```

### 否定形式：`if !consteval`

```cpp
constexpr int f(int n) {
    if !consteval {
        // 运行期路径
        log_call(n); // 只有运行期才做日志
    }
    return n;
}
```

## `if consteval` 与 `if (std::is_constant_evaluated())` 的关键区别

|            | `if (std::is_constant_evaluated())`                          | `if consteval`                                               |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 语法性质   | 普通函数调用 + 普通 `if`                                     | 专用语言语法                                                 |
| 条件表达式 | 可以嵌入更复杂的逻辑（如 `if (cond && std::is_constant_evaluated())`） | **条件只能是纯粹的 `consteval`/`!consteval`，不能和其他表达式组合** |
| 常见误用   | 容易被误用于普通布尔逻辑中，语义边界模糊                     | 语法上强制"这就是一个编译期/运行期分支"，杜绝了误用          |
| 可读性     | 需要理解 `is_constant_evaluated()` 的特殊语义                | 直观地表达意图                                               |

也就是说，`if consteval` **不能**这样写：

```cpp
if consteval && some_condition { ... } // ❌ 编译错误，consteval 不能参与逻辑组合
```

这是有意为之的限制：`if consteval` 被设计成**只能单独作为条件**，从语法层面避免开发者把"是否编译期求值"和其他业务逻辑混在一起，减少误用空间。

## 实际用途举例

### 1. 编译期用简单实现，运行期用高性能/SIMD实现

```cpp
constexpr std::size_t strlen_impl(const char* s) {
    if consteval {
        std::size_t n = 0;
        while (s[n] != '\0') ++n;
        return n;
    } else {
        return std::strlen(s); // 运行期调用高度优化的库函数（可能用 SIMD）
    }
}
```

### 2. 编译期抛异常报错，运行期走正常逻辑

```cpp
constexpr int safe_divide(int a, int b) {
    if consteval {
        if (b == 0) throw "division by zero in constant expression"; 
        // 编译期除零会直接导致编译失败，报错信息更清晰
    }
    return a / b;
}
```

### 3. 结合 `if !consteval` 做运行期专属操作（如断言、日志）

```cpp
constexpr void validate(int x) {
    if !consteval {
        assert(x >= 0); // 运行期才检查断言（编译期违反的话会导致 ill-formed，效果不同）
    }
}
```

## 小结

`if consteval` 是 C++23 对 C++20 `std::is_constant_evaluated()` 使用体验的一次"语法糖化 + 安全加固"：

- 把"区分编译期/运行期执行路径"这一常见需求，从**普通函数 + 普通 if 语句的组合**提升为**专用语言语法**；
- 通过限制条件表达式只能是纯 `consteval`/`!consteval`，**从语法层面消除了误用空间**（不能和其他逻辑混合）；
- 语义更直观，代码意图一目了然，是 `constexpr`/`consteval` 生态在 C++23 里持续完善的一部分（配合前面提到的更多 `constexpr` 标准库支持、`consteval` 传播规则变化等）。