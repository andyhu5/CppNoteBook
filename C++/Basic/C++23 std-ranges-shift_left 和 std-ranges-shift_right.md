`std::ranges::shift_left` 和 `std::ranges::shift_right`

 是 C++23 新增的算法（提案 P2440），用于填补一个历史空缺——C++20 引入了非 ranges 版本的 `std::shift_left`/`std::shift_right`（位于 `<algorithm>`），但当时的 `ranges::` 约束化算法集合里漏掉了这两个，直到 C++23 才补齐。

## 声明（简化版）

```cpp
namespace std::ranges {
    template<std::permutable I, std::sentinel_for<I> S>
    constexpr subrange<I> shift_left(I first, S last, 
                                       std::iter_difference_t<I> n);

    template<ranges::forward_range R>
        requires std::permutable<ranges::iterator_t<R>>
    constexpr ranges::borrowed_subrange_t<R> shift_left(R&& r, 
                                       ranges::range_difference_t<R> n);

    // shift_right 签名类似，但要求 bidirectional_range
}
```

## 功能

**在同一个范围内**，把元素向左（或向右）移动 `n` 个位置，被"挤出"边界的元素值变得不确定，返回值指向移动后**有效数据所在的子范围**。

- `shift_left`：元素整体左移，末尾留出 `n` 个"脏"位置。
- `shift_right`：元素整体右移，开头留出 `n` 个"脏"位置。

这类似于"就地删除头部/尾部一段元素但不改变容器大小"的操作。

## 示例：shift_left

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7};

    auto result = std::ranges::shift_left(v, 3);
    // 把元素左移 3 位

    for (int x : result) std::cout << x << ' ';
    std::cout << '\n'; // 4 5 6 7

    // v 本身的内容变为：{4, 5, 6, 7, ?, ?, ?}（后三个值不确定）
}
```

## 示例：shift_right

```cpp
std::vector<int> v = {1, 2, 3, 4, 5, 6, 7};

auto result = std::ranges::shift_right(v, 3);
// 把元素右移 3 位

for (int x : result) std::cout << x << ' ';
std::cout << '\n'; // 1 2 3 4

// v 本身的内容变为：{?, ?, ?, 1, 2, 3, 4}（前三个值不确定）
```

## 返回值细节

返回的是一个 `subrange`，表示移动后**仍然有效、保持原顺序**的那部分元素：

- `shift_left(r, n)` 返回 `[first, last - n)` 对应的子范围（长度为 `size - n`）
- `shift_right(r, n)` 返回 `[first + n, last)` 对应的子范围

如果 `n <= 0` 或 `n >= size(r)`，则：

- `n <= 0`：不做任何移动，返回整个原范围
- `n >= size(r)`：所有元素都被移出有效区域，返回一个空的子范围（`[last, last)`）

## 与非 ranges 版本 `std::shift_left`/`std::shift_right` 的区别

| 特性     | `std::shift_left/right` (C++20) | `std::ranges::shift_left/right` (C++23)                      |
| -------- | ------------------------------- | ------------------------------------------------------------ |
| 头文件   | `<algorithm>`                   | `<algorithm>`                                                |
| 参数形式 | 仅迭代器对                      | 支持范围重载 + 迭代器/哨兵重载                               |
| 返回类型 | 单个迭代器（新的末尾位置）      | `subrange`（含起止两端）                                     |
| 约束     | 无概念约束                      | `permutable`（left）/ 额外要求 `bidirectional_range`（right） |
| 哨兵支持 | 不支持                          | 支持                                                         |

值得注意的一点不对称：`shift_left` 只需要 `forward_range`，而 `shift_right` 需要 `bidirectional_range`，这是因为右移操作通常需要从末尾往前逐个搬移元素（类似 `move_backward` 的实现方式），前向迭代器无法高效地反向遍历。

## 典型用途

常用于"就地过滤/删除一段元素"的场景，比如实现一个简单的滑动窗口移除逻辑：

```cpp
std::vector<int> buffer = {10, 20, 30, 40, 50};

// 丢弃最旧的 2 个元素，把新数据搬到前面（模拟环形缓冲区腾出空间）
auto valid = std::ranges::shift_left(buffer, 2);
// buffer 前 3 个位置现在是 {30, 40, 50}，可以在后面继续写入新数据
```

## 小结

`ranges::shift_left`/`ranges::shift_right` 是对 C++20 `std::shift_left`/`std::shift_right` 的 ranges 化补齐，提供了范围重载、概念约束和更明确的 `subrange` 返回值，属于 C++23 中"查漏补缺"性质的 ranges 算法之一。