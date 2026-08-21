**C++ DR11（P0588R1）：Simplifying implicit lambda capture** 是 C++20 的一项缺陷报告（Defect Report），它简化了 lambda 表达式中**隐式捕获**（implicit capture）的规则，使其更一致、更易于实现，尤其是在泛型 lambda（generic lambda）和模板上下文中。

### 背景问题

在 C++11 引入 lambda 后，隐式捕获的规则相对复杂，主要基于 **ODR-use**（One Definition Rule use）来决定是否需要捕获变量。

- 如果变量仅在常量表达式上下文中被“加载”（如 `b ? n : 1` 中的 `n`，或 `sizeof`、`decltype` 等 unevaluated 上下文），传统规则可能**不捕获**它。
- 这在**泛型 lambda**（带 `auto` 参数）+ `if constexpr` + 模板的组合下会产生严重问题：编译器为了确定 lambda 的捕获集，必须提前实例化 lambda 体，导致在不应该实例化的分支中也出现错误（例如 `std::unique_ptr` 不可复制的情况）。

示例（简化自提案）：

```cpp
template<typename T>
void f(T t) {
    auto lambda = [&](auto a) {
        if constexpr (/* some condition */) {
            T u = t;  // 可能在某些实例化中不可行
        } else {
            // ...
        }
    };
}
```

旧规则下，编译器需要提前分析捕获，导致问题。

### 新规则（P0588R1 后的简化）

**核心变化**：隐式捕获的判断更**语法化**（syntactic），不再过度依赖复杂的 ODR-use 分析。

- 当一个变量在 lambda 体中被**引用**（且不在 unevaluated operand 如 `sizeof`、`decltype`、`typeid` 中），**只要从变量声明到使用点之间的所有 lambda 都有 capture-default**（`[=]` 或 `[&]`），则该变量会被**隐式捕获**。
- 即使变量仅用于常量表达式上下文，也会形式上捕获（但实现可优化掉未使用的捕获）。
- 这使得 lambda 体的实例化可以**延迟**，解决泛型 lambda + `constexpr if` 的棘手问题。

**cppreference 总结**：新规则下，隐式捕获更“激进”一些，但大幅简化了语言规则和实现难度。

### 实际影响

- **正面**：
  - 改善模板和泛型 lambda 的可用性。
  - 减少实现分歧（不同编译器对旧规则的处理不一致）。
  - 捕获行为更可预测（基于语法结构而非深层语义分析）。
- **潜在 breaking change**：
  - 某些仅用于常量表达式的变量现在会被捕获（可能引入复制，即使是 trivial 的）。
  - 但影响很小：仅限于 `const` 整数/枚举或 `constexpr` 变量，且实现可优化。
- **编译器支持**：GCC/Clang/MSVC 等主流编译器已在 C++20 模式下支持（部分实现可能仍有旧行为残留）。

### 示例对比（来自 cppreference）

```cpp
const int x = 17;

auto g0 = [](auto a) { f(x); };           // 不一定捕获（旧规则）
auto l1 = [=] { f(x); };                  // OK: 捕获 x（新规则）
auto g1 = [=](auto a) { f(x); };          // 类似

auto ltid = [=] { typeid(x); };           // 现在也会捕获（新规则）
```

### 总结

这个 DR 的目标是**让 lambda 隐式捕获规则更简单、更健壮**，避免模板实例化时的复杂性。它是 C++20 lambda 改进的一部分（与其他如 `template` lambda、`[*this]` 等一起）。对于日常编码，影响不大，但对库作者和重度模板使用者非常友好。

想看完整提案可参考：
- [P0588R1](https://wg21.link/P0588R1)
- cppreference 的 Lambda 页面有详细前后对比。