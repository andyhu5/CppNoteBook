# C++23：DR — "consteval Needs to Propagate Up"（P2564R3）

## 背景：C++20 的 `consteval`（立即函数）

C++20 引入了 `consteval` 关键字，声明"**立即函数（immediate function）**"——这类函数**必须**在编译期被求值，任何试图在运行期调用它的代码都是编译错误：

```cpp
consteval int square(int n) {
    return n * n;
}

constexpr int x = square(5); // OK，编译期求值
int y = 10;
int z = square(y);           // ❌ 错误，y 不是编译期常量，无法满足 consteval 要求
```

## 问题：`consteval` 函数被 `constexpr` 函数间接调用时的困境

C++20 存在一个尴尬的场景：如果一个**普通的 `constexpr` 函数**内部调用了一个 `consteval` 函数，而这个 `constexpr` 函数本身**既可能在编译期被调用，也可能在运行期被调用**（`constexpr` 函数的特点就是"可以两种方式调用"），那么就会产生矛盾：

```cpp
consteval int identity(int n) {
    return n;
}

constexpr int f(int n) {
    return identity(n) + 1; // 这里调用了 consteval 函数
}

constexpr int a = f(5);  // OK：f 在编译期被调用，identity(5) 也是编译期求值，没问题
int b = 10;
int c = f(b);             // ❓ f 在运行期被调用，此时 identity(b) 试图在运行期对 consteval 函数求值！
```

在 C++20 的原始规则下，这种写法在**语法层面上是允许通过编译的**——因为 `f` 本身只是 `constexpr`，不是 `consteval`，标准没有强制要求"一个 `constexpr` 函数内部用到了 `consteval` 函数，就必须让这个 `constexpr` 函数本身也变成不能在运行期调用"。但实际执行到 `f(b)` 这一行、`identity(b)` 需要在运行期对一个 `consteval` 函数求值时，这在语义上是矛盾的——**这本质上违反了 `consteval` 的核心保证："这个函数只能在编译期被求值"**。

不同编译器对这种边界情况的处理在 C++20 阶段并不完全一致，标准文字本身也没有把这个"矛盾"讲清楚，导致这属于一个**需要被修复的语言规则漏洞**。

## C++23 的修正：`consteval` 具有"传染性"，需要向上传播

C++23（P2564R3，作为 DR 追溯适用）明确规定：如果一个 `constexpr`（或普通）函数中，**存在某条执行路径必然会调用一个 `consteval` 函数**，且这个调用**不能在编译期完成求值**，那么这种情况需要被正确处理——具体机制是引入了 **"immediate-escalating function"（立即升格函数）** 的概念：

- 如果一个 `constexpr` 函数体内**直接调用**了一个 `consteval` 函数，且该调用的实参**不是**在编译期已知的（无法常量求值），那么这个外层函数会被要求：
  - 要么这条调用路径**必须**在编译期被求值（否则报错）；
  - 要么编译器需要采用更精细的规则来判断该外层函数是否也应该被当作"立即函数"处理。

更准确地说，C++23 引入的核心机制是：**函数调用表达式本身可以成为"立即调用（immediate invocation）"**，当一个函数调用了 `consteval` 函数、且这个调用本身满足特定条件时，这个**调用点会被当作立即函数上下文处理**，如果它出现在一个本应支持运行期调用的普通函数里，且传入的是运行期才知道的值，就会在编译期直接报错，而不是留下一个"看似合法、实际语义矛盾"的漏洞。

```cpp
consteval int identity(int n) {
    return n;
}

constexpr int f(int n) {
    return identity(n) + 1;
}

constexpr int a = f(5);  // OK
int b = 10;
int c = f(b);             // C++23 起：明确报编译错误
                           // （C++20 下这里的行为在不同编译器间可能不一致，或产生令人困惑的结果）
```

## 一个相关的配套规则：`std::is_constant_evaluated()` 与 `consteval` 的交互

这次修正也顺带明确了一些边界情况，比如 `consteval` 函数内部使用 `if consteval` / `std::is_constant_evaluated()` 时的具体语义（因为 `consteval` 函数**总是**在编译期求值，所以 `is_constant_evaluated()` 在其中的结果理应恒为 `true`，标准文字需要精确表述这一点，避免歧义）。

## 为什么这是 DR（缺陷修复）而不是新特性

和之前提到的**lambda 尾置返回类型作用域**、**收窄转换规则**类似，这也被认为是对 **C++20 `consteval` 原始设计的漏洞修复**——标准委员会认为 C++20 里"`consteval` 调用可能被包裹在普通 `constexpr` 函数中、而错误地被允许在运行期上下文里出现"这一行为本身是设计缺陷，值得直接修正规则本身，而不是等待下一个大版本。因此各编译器通常会**追溯性地**在其 C++20 模式下也采用修正后的规则（具体要看编译器版本和实现进度）。

## 实际影响

对大多数日常代码几乎没有影响，因为大部分人很少直接混用 `consteval`/`constexpr` 到这种边界程度。这个修正主要影响：

- 库作者在设计**混合了 `consteval` 和 `constexpr` 的 API**时（比如某些编译期反射、字符串字面量处理、静态断言辅助工具等场景）；
- 需要确保"某个函数的某条调用路径必须强制编译期求值"这类精确控制需求的高级模板/元编程代码。

## 小结

"consteval needs to propagate up" 解决的核心问题是：**C++20 里，`consteval`（立即函数）被 `constexpr`（可编译期可运行期）函数间接调用时，存在语义矛盾却未被规则明确禁止的漏洞**。C++23 通过引入"立即升格函数/立即调用"的精确规则，堵上了这个漏洞，确保 `consteval` 承诺的"只能编译期求值"这一保证，能够**正确地沿调用链向上传播**，而不会在某个中间层的 `constexpr` 函数处被悄悄"稀释"掉——这体现了 C++23 系列 DR 修正一贯的主题：让编译器在规则模糊、可能导致运行期矛盾或未定义行为的边界情况下，尽早在编译期报错，而不是留下一个隐蔽的坑。