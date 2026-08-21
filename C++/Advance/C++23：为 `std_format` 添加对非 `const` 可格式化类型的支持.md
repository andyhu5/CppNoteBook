**C++23：为 `std::format` 添加对非 `const` 可格式化类型的支持**

这是 C++23 中 `<format>` 库的一个重要改进，主要来自提案 **P2418R2 "Add support for non-const-formattable types to std::format"**。

### 1. C++20 的限制

在 C++20 中，`std::format` 对类型的**格式化要求非常严格**：

- 类型 `T` 必须能通过 **`const T&`** 被格式化（即 `formatter<T>` 需要能接受 `const T&`）。
- 这导致很多**非 `const` 类型**（尤其是含有 `mutable` 成员、移动语义、或需要修改状态的类型）**无法直接被格式化**。

**典型问题示例**（C++20 中无法编译或行为受限）：

```cpp
struct Widget {
    int value = 42;
    void update() { value++; }   // 非 const 操作
};

std::format("{}", w);   // 错误或无法正常工作
```

即使 `Widget` 实现了 `formatter`，也经常因为 `const` 限制而失败。

### 2. C++23 的改进

C++23 放宽了这一限制：

- **`std::format` 现在支持 `T` 是非 `const` 的情况**。
- 格式化器（`std::formatter<T>`）现在可以接受**非 `const` 引用**（`T&`）。
- 核心变化：在格式化过程中，传递给格式化器的对象**不再强制为 `const`**。

**关键效果**：

- 允许格式化那些**需要在格式化过程中修改自身状态**的类型。
- 更好地支持**移动语义**、**缓存**、**懒计算**等场景。
- 与 `std::print`、`std::println` 等新设施无缝配合。

### 3. 使用示例

```cpp
struct LazyString {
    std::string data;

    // 非 const 成员函数
    std::string_view get() {
        if (data.empty()) data = "computed value";
        return data;
    }
};

// 在 C++23 中可以直接格式化
LazyString s;
std::cout << std::format("Result: {}", s);   // 合法
```

**自定义 formatter 示例**（支持非 const）：

```cpp
template<>
struct std::formatter<LazyString> {
    constexpr auto parse(std::format_parse_context& ctx) {
        return ctx.begin();
    }

    auto format(LazyString& obj, std::format_context& ctx) const {  // 注意：非 const 引用
        return std::format_to(ctx.out(), "{}", obj.get());
    }
};
```

### 4. 设计细节与注意事项

- **向后兼容**：所有原有 `const` 可格式化类型继续正常工作。
- **不改变 `const` 语义**：如果你传入 `const` 对象，格式化器仍会收到 `const T&`。
- **主要受益场景**：
  - 需要在格式化时计算/缓存结果的类型
  - 移动-only 类型（配合 C++23 其他改进）
  - 具有 `mutable` 成员的类型
- **限制**：仍然不能格式化**无法复制/移动**且没有合适 formatter 的类型。

### 5. 为什么这个改进重要？

- 让 `std::format` **更加灵活**，减少了“为了格式化而被迫增加 `const` 版本函数”的 hack。
- 提升了与现代 C++（移动语义、lazy evaluation）的契合度。
- 是 C++23 对 `<format>` 库进行的一系列可用性修复的一部分（包括编译期检查、二进制体积优化等）。

---

**总结**：

C++23 移除了 `std::format` 对“必须是 `const` 可格式化”的强制要求，允许更多实际类型直接参与格式化，这是对开发者体验的显著提升。

完整测试代码：

```c
struct LazyString {
    std::string data;

    // 非 const 成员函数
    std::string_view get() {
        if (data.empty()) data = "computed value";
        return data;
    }
};

template<>
struct std::formatter<LazyString> {
    constexpr auto parse(std::format_parse_context& ctx) {
        return ctx.begin();
    }

    auto format(LazyString& obj, std::format_context& ctx) const {  // 注意：非 const 引用
        return std::format_to(ctx.out(), "{}", obj.get());
    }
};


int main() {
  // 在 C++23 中可以直接格式化
  LazyString s;
  std::cout << std::format("Result: {}", s) << "\n"; // 合法
}
```

