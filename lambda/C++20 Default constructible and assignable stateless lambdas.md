**C++20: Default constructible and assignable stateless lambdas**（P0624R2）是 C++20 对 lambda 表达式的重要改进之一。

### 背景

在 C++11~C++17 中，**lambda 的 closure type**（闭包类型，即 lambda 生成的唯一未命名类类型）有以下限制：

- **没有默认构造函数**（即使是无捕获的 stateless lambda）。
- **赋值运算符被删除**（deleted）。

这导致即使是简单的 stateless lambda，也无法轻松用于某些需要默认构造或赋值的场景（如标准容器的比较器）。

### C++20 的变化

对于**无捕获**（stateless / captureless）的 lambda：

- **默认构造函数** 被 **defaulted**（默认生成）。
- **拷贝/移动赋值运算符** 也被 **defaulted**。

有捕获（stateful）的 lambda 仍然保持原有行为：**没有默认构造函数**，**赋值运算符被删除**。

**cppreference 总结**：
- 无捕获时，closure type 有 defaulted default constructor 和 defaulted copy/move assignment operators。
- 这些特殊成员函数通常是 **trivial** 的（实现意图如此）。

### 主要动机和用途

1. **与普通 functor 一致性**：
   lambda 本质上是语法糖生成的函数对象，现在 stateless lambda 在使用上更接近手写的空类 functor。

2. **在容器和模板中的直接使用**（最常见场景）：

```cpp
// C++20 前需要 workaround
auto greater = [](auto x, auto y) { return x > y; };
std::map<std::string, int, decltype(greater)> m;  // 错误！无法默认构造

// C++20 后直接可用
std::map<std::string, int, decltype(greater)> m;  // OK
```

结合 **Lambdas in unevaluated contexts**（P0315R4，允许在 `decltype`、`sizeof` 等 unevaluated 上下文中使用 lambda），这个特性非常强大。

3. **库设计**：
   - 便于实现值语义的运行时多态（如 Dyno 库示例）。
   - 更容易在模板元编程或类型擦除场景中使用 lambda 类型。

### 示例

```cpp
// 无捕获的 lambda
auto cmp = [](int a, int b) { return a > b; };

// C++20 前：无法默认构造
// C++20 后：
std::set<int, decltype(cmp)> s;           // OK
std::map<int, int, decltype(cmp)> m;      // OK

auto cmp2 = cmp;   // 赋值也 OK
cmp2 = cmp;        // 赋值 OK
```

**注意**：如果 lambda 有任何捕获（即使是 `[=]` 但实际未捕获变量），默认构造和赋值依然被删除。

### 效果与兼容性

- **不影响现有代码**（purely additive）。
- 极大提升了 lambda 作为“轻量 functor”的实用性，尤其在标准库算法和关联容器中。
- 常与 C++20 其他 lambda 改进（如 implicit capture 简化、template lambdas）一起使用。

这个特性让 lambda 更加“第一公民”，减少了手写空类或使用 `std::function` 等开销较大的替代方案的需求。

想深入了解可参考提案：[P0624R2](https://wg21.link/P0624R2)。