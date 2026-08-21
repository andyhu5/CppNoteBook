**C++23 DR（P2210R2 "Superior String Splitting"）**

对 `views::split` 进行了重大重新设计，并引入了 `views::lazy_split` / `lazy_split_view`。

这是 C++ Ranges 库中一个非常实用的改进，尤其针对字符串分割等常见场景。

### 1. 背景：C++20 原始 `split_view` 的问题

C++20 中的 `std::ranges::views::split`（`split_view`）过于“懒惰”（lazy）：

- 它总是返回 `forward_range`（即使底层是 `contiguous_range` 如 `std::string`）。
- 子范围（inner range）的迭代器类型**不是**底层 view 的迭代器，而是独立的“lazy”实现。
- 无法很好地支持 `bidirectional`、`random_access`、`contiguous` 等强范畴。
- 构造 `std::string`、`std::string_view`、`from_chars` 等操作非常不方便（因为不是 `common_range` 或缺少连续内存）。
- 对字符串分割（最常见用例）体验很差。

### 2. P2210R2 的改动（DR 回溯应用于 C++20）

**主要设计变更**：

1. **重命名 + 分离**：
   - 原有的“超级懒惰”版本 → 改名为 **`lazy_split_view`** / **`views::lazy_split`**（保留 input_range 支持）。
   - 新增一个**更“急切”/更强大的** **`split_view`** / **`views::split`**。

2. **新 `split_view` 的特性**（推荐使用）：
   - **仅支持** `forward_range` 或更强的底层 view（`V`）。
   - 分割结果的子范围是 **`subrange<iterator_t<V>>`**（直接使用底层迭代器）。
   - **保留底层 view 的范畴**：
     - 输入 `contiguous_range` → 输出 `contiguous_range`（子范围是连续的）。
     - 输入 `bidirectional_range` → 输出 `bidirectional_range`。
   - 子范围通常是 `common_range`（便于构造 `string`、`string_view` 等）。
   - 性能更好（提前查找分隔符，利用多趟遍历）。

3. **`lazy_split_view`**（保留旧行为）：
   - 支持 `input_range`（单趟）。
   - 更“懒”，但功能较弱（不保留强范畴）。
   - 适用于 `const` view 或只需要 input_range 的场景。

### 3. 使用示例

```cpp
#include <ranges>
#include <string_view>
#include <iostream>

int main() {
    using namespace std::literals;

    constexpr auto text = "Hello^_^C++^_^20^_^!"sv;
    constexpr auto delim = "^_^"sv;

    // 推荐：新 split_view（C++23 风格）
    for (const auto word : text | std::views::split(delim)) {
        // word 是 subrange<string_view::iterator>，可直接构造 string_view
        std::cout << std::quoted(std::string_view{word}) << ' ';
    }
    // 输出: "Hello" "C++" "20" "!"
}
```

**与 `lazy_split` 的对比**：

- `views::lazy_split`：更通用（支持 input），但子范围弱。
- `views::split`：更强大、更高效，适合大多数字符串/序列分割。

### 4. 其他细节

- **分隔符**：可以是单个元素（`char` 等）或一个 `forward_range`（如 `string_view`）。
- **空子范围**：正确处理尾部空段等边缘情况（修复了 LWG 问题）。
- **性能**：新 `split_view` 在 forward 及以上范畴下明显更优。
- **兼容性**：这是 post-C++20 Defect Report，主流编译器（如 GCC、Clang、MSVC）已在 C++20 模式下回溯实现。

### 总结

P2210R2 是“让字符串分割真正好用”的关键修复：
- **`views::split`**（新版）→ 日常推荐，强范畴、易用。
- **`views::lazy_split`** → 特殊场景（input-only 或 const）备用。

这极大提升了 Ranges 在文本处理、解析等领域的实用性。

如果你需要更多代码对比、实现细节或与其他 view（如 `join`）的配合示例，随时告诉我！