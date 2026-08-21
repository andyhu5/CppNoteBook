C++23 在 `<numeric>` 中新增了若干"约束化"(constrained,即基于 ranges 的) 算法，`std::ranges::iota` 就是其中之一——它是经典 `std::iota`（自 C++11 起就有）的 ranges 版对应物。

## 声明（简化版）

```cpp
namespace std::ranges {
    template<std::input_or_output_iterator O, std::sentinel_for<O> S, 
             std::weakly_incrementable T>
        requires std::indirectly_writable<O, const T&>
    constexpr iota_result<O, T> iota(O first, S last, T value);

    template<std::weakly_incrementable T, std::output_range<const T&> R>
    constexpr iota_result<std::ranges::borrowed_iterator_t<R>, T>
        iota(R&& r, T value);
}
```

它位于头文件 `<numeric>` 中。

## 功能

从 `value` 开始，依次为范围中的每个元素赋以递增的值，每次写入后自增 `value`——核心行为与 `std::iota` 相同。

## 与 `std::iota` 的主要区别

1. **范围重载**——可以直接传入一整个范围（容器、view 等），而不必传迭代器对。

2. **返回有用的信息**——不再是 `void`，而是返回 `iota_result<O, T>`，一个包含两个成员的结构体：

   - `.out` —— 到达的末尾迭代器/哨兵位置
   - `.value` —— 最终值（即最后写入值的下一个值）

   ```
   std::iota
   ```

    无法让你知道最后用到的值是多少（除非自己重新计算），而 

   ```
   ranges::iota
   ```

    直接把它返回给你。

3. **概念约束（concept-constrained）**——要求满足 `weakly_incrementable<T>` 和 `indirectly_writable`，这样能在调用处给出更好的错误信息，并拒绝不合理的类型。

4. **对哨兵友好**——使用 `sentinel_for<O>` 而非要求两个相同类型的迭代器，因此能与以哨兵结尾的范围组合使用。

## 示例

```cpp
#include <numeric>
#include <ranges>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v(5);
    auto result = std::ranges::iota(v, 1);

    for (int x : v) std::cout << x << ' ';   // 1 2 3 4 5
    std::cout << "\n下一个值将是: " << result.value << '\n'; // 6
}
```

使用迭代器/哨兵形式：

```cpp
std::vector<int> v(5);
auto result = std::ranges::iota(v.begin(), v.end(), 10);
// v = {10, 11, 12, 13, 14}, result.value == 15, result.out == v.end()
```

## 背景

它属于 C++23 在 `<numeric>` 中新增的一批约束化数值算法，同批还包括 `std::ranges::fold_left`、`fold_right`、`fold_left_first` 等——是 ranges 项目对 C++20 时 `<algorithm>` 中已覆盖的 `std::ranges::*` 算法集合的补充，填补了 `<numeric>` 部分尚未覆盖的空白。