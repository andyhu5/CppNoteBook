**C++20: Type mismatch of defaulted special member functions**（P0641R2）是一项针对**显式 defaulted 特殊成员函数**（explicitly defaulted special member functions）的语言规则改进，主要解决 `const` 引用等类型不匹配导致的编译错误。

### 问题背景（C++11~C++17）

在 C++ 中，**特殊成员函数**（Special Member Functions）包括默认构造函数、拷贝/移动构造函数、拷贝/移动赋值运算符、析构函数。

当你**显式使用 `= default`** 时，标准要求其**声明的函数类型**必须与编译器**隐式声明**的版本**完全一致**（除少数例外，如 ref-qualifier 和拷贝构造函数/赋值允许 non-const 引用参数）。

**典型问题示例**：

```cpp
struct MyType {
    // 假设 MyType 的拷贝构造函数是 MyType(MyType&) （non-const）
};

struct Wrapper {
    MyType m;
    Wrapper(const Wrapper&) = default;  // C++17 前：错误！
};
```

- `Wrapper` 的**隐式拷贝构造函数**参数类型为 `Wrapper&`（non-const）。
- 但你显式写了 `const Wrapper&`，类型不匹配 → 程序 **ill-formed**（编译错误），即使这个拷贝构造函数从未被实际调用。

类似问题也出现在 `std::tuple`、`std::pair` 等包装类中，当成员类型的特殊成员函数签名有细微差异时，会导致整个包装类无法正常使用。

### C++20 的解决方案

**P0641R2** 修改规则：

- 如果显式 `= default` 的函数**类型与隐式版本不匹配**（在允许的例外之外），**不再是 ill-formed**。
- 而是**将该函数定义为 deleted**（就像 `= delete` 一样）。

这样：

- 代码可以编译通过。
- 只有在**实际尝试使用**该特殊成员函数时，才会因为它是 deleted 而报错。
- 提高了可用性，尤其对 wrapper 类、聚合类型和泛型代码非常友好。

### 效果对比

**C++17 前**：
- 类型轻微不匹配 → 整个类无法使用（即使不拷贝）。

**C++20 后**：
- 类型不匹配 → 函数被 deleted。
- 类本身可用，只要不触发 deleted 的特殊成员函数。

这与用户直觉更一致：`= default` 是“请求默认实现”，如果无法精确匹配，就退化为“禁用”而不是直接拒绝整个类型。

### 相关 Core Issue

该提案主要解决了 **CWG 1331**（const mismatch with defaulted copy constructor）和其他相关问题。

### 实际意义

- 提升了**库代码**的鲁棒性（尤其是 `std::tuple` 等）。
- 让用户在 wrapper 类中更灵活地声明 `const` 版本的拷贝/移动函数。
- 属于 C++20 对特殊成员函数规则的“清理”改进之一，与 `P0624R2`（stateless lambda）等特性共同提升语言一致性。

这个变化是向后兼容的，不会破坏现有正确代码，仅放宽了之前过于严格的限制。

想查看完整提案可参考：[P0641R2](https://wg21.link/P0641R2)。如果需要代码示例或与其他 C++20 特性的对比，随时告诉我！