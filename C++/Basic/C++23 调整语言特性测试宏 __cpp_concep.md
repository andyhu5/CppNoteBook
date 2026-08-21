**C++23 中的 DR20（Defect Report 20）** 是关于调整语言特性测试宏 **`__cpp_concepts`** 值的修正。

### 背景
C++20 引入了 **Concepts**（概念）特性，`__cpp_concepts` 宏用于检测编译器对概念的支持程度。其初始值为 **201907L**（对应 C++20 核心 Concepts 提案）。

随后有一些重要的 C++20 改进影响了 Concepts：
- **P0848R3**：允许**条件平凡（conditionally trivial）的特殊成员函数**（例如在某些条件下 `special member functions` 可以是 trivial 的）。

P2493R0 提案（即 DR20）认为这些改进属于 Concepts 特性的“修订”，因此应该更新宏的值以反映完整的 C++20 Concepts 支持。

### 调整内容
- 将 `__cpp_concepts` 的值从 **201907L** **提升到 202002L**。
- 这个新值代表**包含条件平凡特殊成员函数的完整 Concepts 支持**。

在 cppreference 的特性测试表中可以看到：
- **201907L**：基础 Constraints（C++20）
- **202002L**：Conditional trivial special member functions（C++20，通过 DR20 调整）

### 为什么需要调整？
特性测试宏的设计原则是：当特性发生**显著修订**但仍属于同一特性时，会更新宏的值。这样用户代码可以更精确地检测编译器实现了哪些改进（例如使用 `#if __cpp_concepts >= 202002L`）。

这个 DR 被**追溯应用**到 C++20/C++23，确保兼容性和正确性。许多编译器（如 Clang）已据此更新了宏值。

### 实际影响
- **代码兼容**：依赖 `__cpp_concepts` 值的库（如 libstdc++ 的 `<expected>` 等）现在可以正确检测完整支持。
- **条件编译**：你可以这样写：
  ```cpp
  #if __cpp_concepts >= 202002L
  // 使用支持条件平凡特殊成员函数的 Concepts 特性
  #else
  // 回退或警告
  #endif
  ```

这个 DR 是 C++ 标准化中常见的“事后修正”机制，确保标准与实际实现和用户期望保持一致。更多细节可参考 P2493R0 提案和 cppreference 的 [feature test](https://en.cppreference.com/w/cpp/feature_test) 页面。