**C++20 `std::is_bounded_array` 和 `std::is_unbounded_array` 完整介绍**

这两个类型特性（type traits）定义在头文件 **`<type_traits>`** 中，是 C++20 对数组类型进行更精细区分的重要补充。

---

### 1. 作用

C++20 之前，`std::is_array<T>` 只能判断一个类型是否为**数组**，但无法区分**有界数组**和**无界数组**。

C++20 增加了两个更精确的 trait：

| 类型特性                     | 含义                         | 示例                   |
| ---------------------------- | ---------------------------- | ---------------------- |
| `std::is_bounded_array<T>`   | T 是**有界数组**（大小已知） | `int[10]`、`double[5]` |
| `std::is_unbounded_array<T>` | T 是**无界数组**（大小未知） | `int[]`、`char[]`      |

---

### 2. 使用示例

```cpp
#include <type_traits>
#include <iostream>

int main() {
    using A1 = int[10];
    using A2 = int[];
    using A3 = int;

    std::cout << std::boolalpha;

    std::cout << "int[10] is bounded array: " 
              << std::is_bounded_array_v<A1> << '\n';        // true

    std::cout << "int[] is unbounded array: " 
              << std::is_unbounded_array_v<A2> << '\n';      // true

    std::cout << "int is array: " 
              << std::is_array_v<A3> << '\n';                // false

    // 组合使用
    std::cout << "int[10] is array: " 
              << std::is_array_v<A1> << '\n';                // true
}
```

---

### 3. 详细特性

- **`std::is_bounded_array<T>::value`**（或 `_v` 别名）：
  - 当 `T` 是形如 `U[N]`（N 是编译期常量）的数组时返回 `true`。
  - 多维数组也支持（如 `int[3][5]`）。

- **`std::is_unbounded_array<T>::value`**：
  - 当 `T` 是形如 `U[]` 的数组时返回 `true`。
  - 常用于函数参数（如 `void foo(int arr[])`）或 `new` 动态分配场景。

**注意**：
- 引用、指针、函数等**不是**数组。
- `std::is_array_v<T>` = `is_bounded_array_v<T> || is_unbounded_array_v<T>`。

---

### 4. 实际应用场景

1. **模板元编程**中对数组类型的特殊处理：

```cpp
template<typename T>
void process(T& arr) {
    if constexpr (std::is_bounded_array_v<T>) {
        // 已知大小，可以用 std::size(arr)
    } else if constexpr (std::is_unbounded_array_v<T>) {
        // 无界数组，需要额外传递长度
    }
}
```

2. **完美转发 + 数组**：

```cpp
template<typename T>
void wrapper(T&& arr) {
    if constexpr (std::is_unbounded_array_v<std::remove_reference_t<T>>) {
        std::cout << "Unbounded array\n";
    }
}
```

3. **与 `std::extent`、`std::rank` 等数组 trait 配合使用**。

---

### 5. 总结对比

| Trait                     | 有界数组 `int[10]` | 无界数组 `int[]` | 普通类型 `int` |
| ------------------------- | ------------------ | ---------------- | -------------- |
| `std::is_array`           | true               | true             | false          |
| `std::is_bounded_array`   | **true**           | false            | false          |
| `std::is_unbounded_array` | false              | **true**         | false          |

---

这些 trait 在编写**泛型代码**、**数组专用模板**、**内存管理**等场景中非常有用，尤其当你需要对不同数组类型做出不同行为时。

需要我给出更多实际模板示例吗？