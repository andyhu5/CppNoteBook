**C++20 `template-parameter-list` for Generic Lambdas 完整介绍**

C++20 允许在 **lambda 表达式** 中直接声明**显式模板参数**，这使得 generic lambda 变得更加清晰、强大和灵活。

---

### 1. 语法

```cpp
auto lambda = []<template-parameters>(parameters) -> return_type { ... };
```

**关键变化**：在捕获列表 `[]` 之后、参数列表 `()` 之前，增加了 **`<...>`** 模板参数列表。

---

### 2. 示例对比

#### C++17 及之前的写法（隐式模板）

```cpp
auto print = [](const auto& x) {
    std::cout << x << '\n';
};

print(42);
print("hello");
print(3.14);
```

#### C++20 显式模板参数写法（推荐）

```cpp
auto print = []<typename T>(const T& x) {
    std::cout << x << '\n';
};
```

---

### 3. 完整示例

```cpp
#include <iostream>
#include <vector>
#include <string>

int main() {
    // 1. 基本用法
    auto print = []<typename T>(const T& value) {
        std::cout << value << '\n';
    };

    print(100);
    print("C++20");
    print(3.14159);

    // 2. 多个模板参数
    auto multiply = []<typename T, typename U>(T a, U b) -> decltype(a * b) {
        return a * b;
    };

    std::cout << multiply(10, 2.5) << '\n';   // 25

    // 3. 带约束（Concepts，C++20）
    auto process = []<std::integral T>(T x) {
        std::cout << "Integer: " << x << '\n';
    };

    process(42);
    // process(3.14);  // 编译错误：不满足 std::integral
}
```

---

### 4. 重要特性

- **模板参数可以有默认值**：
  ```cpp
  auto func = []<typename T = int>(T value) { ... };
  ```

- **支持非类型模板参数**：
  ```cpp
  auto repeat = []<int N>(const auto& x) {
      for (int i = 0; i < N; ++i) std::cout << x;
  };
  repeat<5>("*");   // 输出 *****
  ```

- **支持 `auto` 参数 + 模板参数混合**（更强大）：
  ```cpp
  auto advanced = []<typename T>(std::vector<T>& vec, auto&& value) {
      vec.push_back(std::forward<decltype(value)>(value));
  };
  ```

---

### 5. 主要优势

| 优势               | 说明                               |
| ------------------ | ---------------------------------- |
| **可读性**         | 模板参数显式可见，比 `auto` 更清晰 |
| **可约束**         | 更容易配合 Concepts 使用           |
| **灵活性**         | 支持非类型模板参数、默认值等       |
| **调试与错误信息** | 编译器错误提示更清晰               |
| **高级元编程**     | 便于编写更复杂的泛型 lambda        |

---

### 6. 注意事项

- 这个特性**只适用于 generic lambda**（即至少有一个 `auto` 参数的 lambda）。
- 普通 lambda（没有 `auto` 参数）不能加模板参数列表。
- 在 C++20 之前，如果需要模板 lambda，通常要借助 `std::function` 或手动定义 functor 类，现在可以直接用 lambda 实现。

---

**总结**：

C++20 允许在 lambda 中直接写 `[]<typename T>(T x){...}`，这是对 **Generic Lambda** 的重大增强，使 lambda 的泛型能力接近普通模板函数。

**推荐**：在新代码中，**优先使用显式模板参数列表**的写法，尤其在需要 Concepts 约束或编写库代码时。

---

需要我继续讲解它与 Concepts 结合的高级用法、或者与 `std::invoke`、`std::function` 的配合吗？