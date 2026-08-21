**C++23 中的 "Using unknown pointers and references in constant expressions"** 是提案 **P2280R4**（对应 **DR20**），显著扩展了 **`constexpr`** 的能力。

### 背景
在 C++20 及更早版本中，**常量表达式**（constant expressions，用于 `constexpr`、`static_assert`、`template` 参数等）对指针和引用的限制非常严格：

- 只能使用**指向编译时已知对象**的指针/引用。
- 不能对**未知地址**（如函数参数、动态分配、外部变量等）的指针/引用进行操作，即使不解引用也不行。
- 这导致许多看似“安全”的代码无法在 `constexpr` 上下文中使用。

例如，以下代码在 C++20 中**不合法**（即使不实际解引用）：

```cpp
constexpr int f(const int* p) {
    if (p == nullptr) return 0;   // 比较未知指针
    return *p;                    // 实际使用时仍受限
}

constexpr int g() {
    int x = 42;
    return f(&x);                 // &x 在 constexpr 中是 "unknown"
}
```

编译器会抱怨指针/引用不是常量表达式。

### C++23 的变更（P2280R4）
提案放松了**常量表达式中对未知指针和引用的限制**，只要**不进行非法操作**（如解引用无效地址、越界访问等）即可。

#### 主要允许的操作（现在在 constexpr 中合法）：
- **指针/引用的比较**（`==`, `!=`, `<`, `>`, `<=`, `>=`）——即使指向未知对象。
- **指针/引用的地址计算**（`&`、`std::addressof`）。
- **指针的算术运算**（在数组内有限制）。
- **将指针/引用作为值传递、存储在 `constexpr` 变量中**。
- **在 `if constexpr` 或 `static_assert` 中使用**。

**关键限制仍然存在**（防止 UB）：
- **不能解引用**指向未知/无效内存的指针（除非保证指向有效对象）。
- **不能访问**对象的值，除非该对象在常量求值期间是“已知”的（live）。
- 数组越界、悬空指针等仍为 ill-formed 或 UB。

#### 示例（C++23 中合法）

```cpp
constexpr int foo(const int* p) {
    if (p == nullptr) return 0;
    return *p;  // 只有在 p 指向有效 constexpr 对象时才安全
}

constexpr int bar() {
    int arr[5] = {1,2,3,4,5};
    const int* p = arr + 2;   // 指针算术
    if (p > arr) {            // 指针比较
        return *p;            // 3
    }
    return 0;
}

static_assert(bar() == 3);    // OK in C++23
```

另一个常见场景：**在 `constexpr` 函数中处理函数参数的指针**：

```cpp
constexpr bool is_null(const int* p) {
    return p == nullptr;      // 现在允许
}

constexpr int value_or(const int* p, int def) {
    return p ? *p : def;
}
```

### 动机与好处
- 让更多**泛型/库代码**能在 `constexpr` 中工作（例如容器、算法、智能指针的 `constexpr` 版本）。
- 减少对 `std::integral_constant`、`std::bool_constant` 等模板 hack 的依赖。
- 使 `constexpr` 更接近运行时代码的表达能力，同时保持安全性。
- 标准化了许多编译器早已支持的扩展行为。

### 实现状态
- **GCC**、**Clang**、**MSVC** 等主流编译器已在 C++23 模式下支持（部分早期支持）。
- 这是 C++23 中对 `constexpr` 的重要增强之一，与 `constexpr` 算法、字符串处理等特性配合良好。

### 总结
这个 DR20/P2280R4 **大幅提升了 `constexpr` 中指针和引用的可用性**，让常量求值更实用，同时严格保留了 UB 防护。开发者现在可以编写更灵活的 `constexpr` 代码，而无需担心“未知地址”限制。

更多细节可参考：
- 提案 [P2280R4](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2280r4.html)
- cppreference 的 [Constant expressions](https://en.cppreference.com/w/cpp/language/constant_expression) 页面（C++23 更新部分）。

如果你有具体代码示例或想对比 C++20 行为，随时告诉我！