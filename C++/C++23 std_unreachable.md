# C++23 *std::unreachable*

C++23 *std::unreachable*是一种将未定义行为注入程序的工具。

典型用例是在 else 分支或 switch 的默认情况中。

编译器可以利用这种未定义的行为并优化相应的分支。

*虽然我们可以使用[[assume(false)]]*实现相同的行为，但*std::unreachable*提供了一个更易读、更合乎逻辑的名称。

```C++
#include <utility>

__attribute__((noinline)) int fun1(int x) {
    switch (x) {
        case 1: return 0;
        case 2: return 1;
        case 4: return 2;
        case 8: return 3;
    }
    return -1;
}

__attribute__((noinline)) int fun2(int x) {
    switch (x) {
        case 1: return 0;
        case 2: return 1;
        case 4: return 2;
        case 8: return 3;
        default: std::unreachable();
    }
    return -1;
}

__attribute__((noinline)) int fun3(int x) {
    return x / 4;
}

__attribute__((noinline)) int fun4(int x) {
    if (x < 0) std::unreachable();
    // same as [[assume(x < 0)]];
    return x / 4;
}


```

这段代码展示了 C++ 中与函数优化和未定义行为相关的特性，特别是 C++23 引入的 std::unreachable()（来自 <utility> 头文件），并结合了 __attribute__((noinline)) 来控制编译器优化。代码定义了四个函数 fun1、fun2、fun3 和 fun4，通过不同的方式处理输入，并使用 std::unreachable() 提示编译器某些代码路径不可达。以下是逐步解释。

------

代码概览

- 使用 __attribute__((noinline)) 阻止函数内联，便于观察编译器行为。
- 定义四个函数，展示 std::unreachable() 的用法及其对优化的影响。

------

关键组件

1. **头文件**

cpp

```cpp
#include <utility>
```

- <utility>：提供 std::unreachable()（C++23），标记不可达代码。
- **fun1：基本 switch**

cpp

```cpp
__attribute__((noinline)) int fun1(int x) {
    switch (x) {
        case 1: return 0;
        case 2: return 1;
        case 4: return 2;
        case 8: return 3;
    }
    return -1;
}
```

- **行为**：
  - 输入 x 为 1, 2, 4, 8 时，返回对应的值（0, 1, 2, 3）。
  - 其他值返回 -1。
- **特点**：
  - 没有假设输入范围，显式处理所有情况。
- **编译器优化**：
  - 由于未标记不可达路径，编译器不会假设 switch 覆盖所有可能输入。
- **fun2：使用 std::unreachable() 的 switch**

cpp

```cpp
__attribute__((noinline)) int fun2(int x) {
    switch (x) {
        case 1: return 0;
        case 2: return 1;
        case 4: return 2;
        case 8: return 3;
        default: std::unreachable();
    }
    return -1;
}
```

- **std::unreachable()**：
  - 告诉编译器 default 分支不可达。
  - 若执行到此处，行为未定义（UB）。
- **行为**：
  - 与 fun1 相同，但假设输入仅为 1, 2, 4, 8。
  - 若 x 不在这些值中，程序进入 UB。
- **优化**：
  - 编译器可假设 switch 覆盖所有合法输入，优化掉 default 分支检查。
  - return -1 不可达，可能被移除。
- **fun3：简单除法**

cpp

```cpp
__attribute__((noinline)) int fun3(int x) {
    return x / 4;
}
```

- **行为**：
  - 对输入 x 除以 4，返回结果。
  - 无任何假设，适用于所有 int 值。
- **特点**：
  - 未使用 std::unreachable()，编译器不会做额外假设。
- **优化**：
  - 仅执行基本算术优化（如用位移替代除法）。
- **fun4：带 std::unreachable() 的条件**

cpp

```cpp
__attribute__((noinline)) int fun4(int x) {
    if (x < 0) std::unreachable();
    return x / 4;
}
```

- **std::unreachable()**：
  - 假设 x < 0 不可达。
  - 若 x < 0，行为未定义。
- **行为**：
  - 假设 x >= 0，返回 x / 4。
- **与 [[assume]] 的比较**：
  - 注释提到等价于 [[assume(x >= 0)]]（C++23 属性）。
  - 两者都提示编译器优化，但 std::unreachable() 是运行时标记 UB，而 [[assume]] 是编译期提示。
- **优化**：
  - 编译器可消除 if 检查，假设 x >= 0，优化为直接 x / 4。

------

为什么这样工作？

1. **std::unreachable()**：
   - 标记代码路径不可达，若执行到此则 UB。
   - 允许编译器假设这些路径不会发生，进行激进优化。
2. **__attribute__((noinline))**：
   - 阻止内联，确保函数独立，便于观察汇编或行为。
3. **优化影响**：
   - fun1 和 fun3：无假设，保守处理。
   - fun2 和 fun4：使用 std::unreachable()，提示编译器优化掉不可达分支。

------

示例行为

- **fun1(4)**：返回 2。
- **fun2(4)**：返回 2；若 fun2(5)，UB。
- **fun3(-4)**：返回 -1。
- **fun4(-4)**：UB（应避免调用）。

------

输出

- 无 main 函数，无运行时输出。
- 代码仅展示编译期行为。

------

使用场景

- **优化**：
  - 使用 std::unreachable() 提示编译器删除死代码。
- **契约编程**：
  - 表示函数输入的隐式约束。
- **调试**：
  - 配合断言验证假设。

------

总结

- fun1：基本 switch，无优化假设。
- fun2：使用 std::unreachable()，假设输入受限。
- fun3：简单除法，无约束。
- fun4：假设 x >= 0，优化除法。
- 代码展示了 std::unreachable() 的作用和优化潜力。