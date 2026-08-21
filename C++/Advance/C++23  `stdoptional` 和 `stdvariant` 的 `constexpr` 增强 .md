**C++23 中 `std::optional` 和 `std::variant` 的 `constexpr` 增强（P2231R1）**

这是 C++23 中一个重要的库改进，让 `std::optional<T>` 和 `std::variant<Ts...>` 在**常量表达式（constexpr）上下文**中更加可用。

### 1. 背景：C++20 的限制

在 C++20 中，`std::optional` 和 `std::variant` 的大部分操作**不是 `constexpr`**，这导致在编译期（`constexpr` 函数、模板元编程、常量初始化等）中使用时非常受限。

例如，以下代码在 C++20 中**无法编译**（或只能部分工作）：

```cpp
constexpr std::optional<int> o = 42;
constexpr auto v = o.value();        // 不行
constexpr std::variant<int, double> var = 3.14;
```

### 2. C++23 的主要改进（P2231R1）

该提案将大量成员函数标记为 `constexpr`，并修复了一些内部实现问题，使这两个类型在常量求值中“基本完整可用”。

#### **std::optional<T> 的 constexpr 增强**

新增/改为 `constexpr` 的关键操作包括：

- **构造函数**：默认构造、从 `T` 构造、从 `std::nullopt` 构造、拷贝/移动构造等。
- **赋值**：`operator=`, `emplace()`, `reset()`。
- **访问**：
  - `operator*()`, `operator->()`
  - `value()`, `value_or()`
  - `has_value()`
- **比较**：`==`, `!=`, `<`, `<=`, `>`, `>=`（以及与 `nullopt` 和 `T` 的比较）。
- **其他**：`swap()`, `transform()`（monadic 操作）等。

**示例**（C++23）：

```cpp
constexpr std::optional<int> foo() {
    std::optional<int> o = 42;
    o = std::nullopt;
    o.emplace(100);
    if (o) {
        return *o + 1;
    }
    return std::nullopt;
}

static_assert(foo() == 101);
```

#### **std::variant<Ts...> 的 constexpr 增强**

- **构造函数**：默认构造、从 `T` 构造、拷贝/移动、in_place 构造等。
- **赋值**：`operator=`, `emplace<Index>()` / `emplace<T>()`。
- **访问**：
  - `std::get<Index>()`, `std::get<T>()`
  - `std::visit()`（有限支持，常量求值中可用）。
- **查询**：`index()`, `valueless_by_exception()`, `holds_alternative<T>()`。
- **比较运算符**。

**示例**（C++23）：

```cpp
constexpr std::variant<int, std::string> make() {
    std::variant<int, std::string> v = 42;
    v.emplace<1>("hello");
    return v;
}

constexpr auto var = make();
static_assert(std::holds_alternative<std::string>(var));
```

### 3. 主要限制（仍存在）

即使在 C++23 中，也**不是完全无限制**的 `constexpr`：

- `std::variant` 的 `std::visit` 在某些复杂情况下仍有局限（尤其是 lambda 捕获或非常量 visitor）。
- 涉及动态分配或异常的场景仍受限（`constexpr` 动态分配在 C++20+ 已部分支持，但仍有规则）。
- `T` 自身必须支持 `constexpr`（例如自定义类型需要 `constexpr` 构造函数/成员函数）。

### 4. 实际意义

- **编译期计算**更强大：可以在 `constexpr` 函数中安全使用 `optional` 作为“可选返回值”，`variant` 作为“类型安全的 union”。
- **更好的模板元编程**和常量初始化（`constinit`）。
- 与 C++23 其他特性（如 `std::expected`、`constexpr` 算法）配合更自然。
- 减少运行时开销：在编译期能完成的工作不再需要运行时执行。

### 5. 相关提案和 DR

- **P2231R1**：核心提案（“Missing constexpr in std::optional and std::variant”）。
- 它是 C++23 的重要“补齐”特性，让标准库的这些常用类型跟上语言的 `constexpr` 能力。

**总结**：C++23 让 `std::optional` 和 `std::variant` 真正成为了**编译期友好**的类型，大幅提升了它们在现代 C++ 中的可用性，尤其适合需要强常量求值的场景。

如果你需要更多具体 API 的 `constexpr` 支持列表、与 C++20 的差异对比代码，或与 `std::expected` 的配合示例，随时告诉我！