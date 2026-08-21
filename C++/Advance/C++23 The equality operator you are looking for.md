**C++23 中的 "The equality operator you are looking for"** 是提案 **P2468R2**（对应 **DR20**），它修复了 C++20 **三路比较（Spaceship Operator）** `operator<=>` 引入后，**相等比较（`==`）** 在某些场景下的意外行为。

### 背景问题（C++20 的痛点）
C++20 引入了 `operator<=>`（三路比较），并对**相等比较**进行了重写规则（rewrite rules）：

- 如果一个类型定义了 `operator<=>`，编译器会自动生成 `operator==`（除非已手动定义）。
- 然而，在**重载解析**和**候选函数**选择时，**`==` 的重写规则**有时会产生**不符合直觉**的结果：
  - 某些情况下，本应选择用户定义的 `==` 却选择了由 `<=>` 生成的版本（或反之）。
  - 导致**二义性（ambiguity）** 或 **意外的相等语义**。
  - 特别是在涉及**继承、混合比较**、**模板**或**用户定义比较**时，行为容易令人困惑。

提案的标题 *"The equality operator you are looking for"* 幽默地表达了：“这就是你真正想要的 `==` 行为”。

### 提案的主要变更（P2468R2）
1. **调整 `==` 的重写规则优先级**：
   - 明确**用户手动定义的 `operator==`** 在重载解析中具有更高优先级。
   - 减少由 `<=>` 自动生成的 `==` 对现有代码的干扰。

2. **改进候选函数集**：
   - 细化何时使用 `<=>` 重写 `==`，何时保留原有 `==`。
   - 解决**对称性**和**一致性**问题（例如 `a == b` 和 `b == a` 的行为）。

3. **修复边缘情况**：
   - 处理涉及**基类/派生类**、**不同类型混合比较**等场景。
   - 使 `==` 的行为更符合程序员的“直觉期望”。

**结果**：`operator==` 的查找和选择更**可预测**、**少二义性**，同时保留了三路比较带来的好处（例如只需定义 `<=>` 即可获得所有比较运算符）。

### 实际影响
- **向后兼容**：大多数现有 C++20 代码不受影响，但修复了一些**隐藏的 bug** 或 **二义性**。
- **更好的体验**：
  - 类中定义了 `operator<=>` 后，`==` 的行为更可靠。
  - 库作者和泛型代码受益（例如 Ranges、Containers）。
- **DR 性质**：作为 Defect Report 追溯应用到 C++20/C++23，确保标准与实际意图一致。

### 示例（简化）
**C++20 可能的问题**（简化场景）：

```cpp
struct A {
    bool operator==(const A&) const;  // 用户定义
    auto operator<=>(const A&) const = default;
};

struct B : A {};

bool f(B& x, B& y) {
    return x == y;  // C++20 可能有意外的重写行为
}
```

**C++23**：`==` 更倾向于使用用户明确定义的版本，行为更直观。

### 总结
P2468R2 是 C++20 **Spaceship Operator** 引入后的**重要修复**，它让**相等比较**回归“程序员真正想要”的行为——**明确、可预测、少坑**。这是 C++23 中众多 DR20 之一，体现了标准化委员会对可用性和一致性的持续打磨。

更多细节可参考：
- 提案 [P2468R2](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2468r2.html)
- cppreference 的 C++23 变更表（DR20 部分）。



测试：

```c
#include <iostream>

struct A {
    bool operator==(const A&) const { std::cout << "user defined function == call()"; return true; }  // 用户定义
    auto operator<=>(const A&) const = default;
};

struct B : A {};

bool f(B& x, B& y) {
    return x == y;  // C++20 可能有意外的重写行为
}

int main() {
	std::cout << __cpp_concepts << '\n'; // 202002L

	B x, y;
	f(x, y);  // 调用 A::operator==，而不是默认的 operator<=>

	return 0;
}
```

