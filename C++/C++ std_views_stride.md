# std::views::stride  (每隔 step - 1 个元素取一个)

以下是 std::views::stride 的完整解释，包含补充内容：

------

定义

std::views::stride 是 C++23 引入的 Ranges 视图适配器，定义在 <ranges> 头文件中。它从一个输入范围生成新视图，视图中的元素按指定步长（stride）从原始范围中选取。

cpp

```cpp
#include <ranges>

namespace std::views {
    inline constexpr auto stride = /* unspecified */;
}
```

- **语法**：

  cpp

  ```cpp
  auto view = std::views::stride(range, step);
  ```

- **range**: 输入范围（满足 std::ranges::input_range）。

- **step**: 步长（正整数）。

- 返回类型：std::ranges::stride_view。

------

行为

- **功能**：
  - 从 range 的第一个元素开始，每隔 step - 1 个元素取一个。
  - 如果原始范围有 n 个元素，步长为 s，**视图大小为 ceil(n / s)。**
- **惰性求值**：
  - 不复制数据，仅在迭代时计算。
- **迭代顺序**：
  - 保持原始顺序，但跳过未选取的元素。

------

前提条件

- **范围**：
  - 必须是 std::ranges::input_range。
  - 随机访问范围（如 std::vector）效率更高。
- **步长**：
  - step > 0，否则未定义行为。

------

示例代码

示例 1：基本步长

cpp

```cpp
#include <ranges>
#include <iostream>
#include <vector>

int main() {
    std::vector<int> vec = {0, 1, 2, 3, 4, 5, 6, 7, 8};

    auto strided = std::views::stride(vec, 3); // 步长 3

    for (int x : strided) {
        std::cout << x << " ";
    }
    std::cout << "\n";

    return 0;
}
```

输出

```text
0 3 6
```

- **解释**：取索引 0, 3, 6（每隔 2 个元素）。

示例 2：与 iota 结合

cpp

```cpp
#include <ranges>
#include <iostream>

int main() {
    auto numbers = std::views::iota(0, 10); // {0, 1, ..., 9}
    auto strided = std::views::stride(numbers, 2);

    for (int x : strided) {
        std::cout << x << " ";
    }
    std::cout << "\n";

    return 0;
}
```

输出

```text
0 2 4 6 8
```

示例 3：多视图组合

cpp

```cpp
#include <ranges>
#include <iostream>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    auto view = vec | std::views::filter([](int x) { return x % 2 == 0; }) // 偶数
                    | std::views::stride(2);                        // 步长 2

    for (int x : view) {
        std::cout << x << " ";
    }
    std::cout << "\n";

    return 0;
}
```

输出

```text
2 6 10
```

- **解释**：
  - 先过滤偶数：{2, 4, 6, 8, 10}。
  - 再步长 2：{2, 6, 10}。

------

返回视图的类型

- **std::ranges::stride_view<R>**：
  - R 是输入范围类型。
- **迭代器**：
  - begin()：第一个元素。
  - end()：结束位置（可能不含最后一个元素）。
- **元素类型**：
  - 与输入范围的元素类型相同。

------

时间复杂度

- **构造**：O(1)。
- **迭代**：
  - 随机访问迭代器：O(1) 每次前进。
  - 输入迭代器：O(step)（需跳跃）。
- **总元素数**：O(n / step)。

------

使用场景

1. **数据采样**：

   - 每隔若干元素取样：

     cpp

     ```cpp
     auto samples = std::views::stride(data, 5);
     ```

2. **矩阵操作**：

   - 在一维数组中按步长访问行/列：

     cpp

     ```cpp
     std::vector<int> matrix = {1, 2, 3, 4, 5, 6}; // 2x3
     auto col0 = std::views::stride(matrix, 3); // {1, 4}
     ```

3. **时间序列**：

   - 按固定间隔处理数据。

------

注意事项

1. **C++23 要求**：
   - 需要 -std=c++23 和支持 C++23 的编译器（GCC 13+、Clang 16+、MSVC 19.31+）。
2. **步长**：
   - step <= 0 是未定义行为。
3. **空范围**：
   - 输入为空，结果视图为空。
4. **性能**：
   - 非随机访问范围的步进成本较高。

------

实现（概念性）

std::views::stride 的核心是一个视图类，迭代器大致如下：

cpp

```cpp
template <std::ranges::input_range R>
class stride_view : public std::ranges::view_interface<stride_view<R>> {
    R base_;
    std::size_t step_;
public:
    class iterator {
        std::ranges::iterator_t<R> current_;
        std::size_t step_;
    public:
        iterator operator++() {
            std::ranges::advance(current_, step_);
            return *this;
        }
        auto operator*() const { return *current_; }
    };
    // 其他成员...
};
```

------

与其他视图的组合

- **过滤后步长**：

  cpp

  ```cpp
  auto v = vec | std::views::drop(1) | std::views::stride(2);
  ```

- **变换后步长**：

  cpp

  ```cpp
  auto v = vec | std::views::transform([](int x) { return x * x; }) | std::views::stride(2);
  ```

------

总结

std::views::stride 是 C++23 中一个灵活的 Ranges 工具，用于按步长提取元素。它提供声明式、惰性求值的接口，适用于采样、矩阵访问等场景。需要 C++23 支持，性能依赖迭代器类型。如果你有具体问题（如与某个范围组合、性能分析），请告诉我，我会深入探讨！