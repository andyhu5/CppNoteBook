ranges::copy应用

不完全对——**单纯的 `ranges::copy` 不会做条件筛选**，它只是无条件地把整个范围复制过去。如果要"只复制满足条件的元素"，有两种标准做法：

## 方案一：`ranges::copy_if`（专门为此设计）

```cpp
namespace std::ranges {
    template<input_range R, weakly_incrementable O, class Pred, class Proj = identity>
        requires indirectly_copyable<iterator_t<R>, O> &&
                 indirect_unary_predicate<Pred, projected<iterator_t<R>, Proj>>
    constexpr copy_if_result<borrowed_iterator_t<R>, O>
        copy_if(R&& r, O result, Pred pred, Proj proj = {});
}
```

用法：

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    std::vector<int> evens;

    std::ranges::copy_if(v, std::back_inserter(evens), 
        [](int x) { return x % 2 == 0; });

    for (int x : evens) std::cout << x << ' '; 
    // 2 4 6 8 10
}
```

这是最直接、最常见的写法，逻辑清晰，一步到位。

## 方案二：`views::filter` + `ranges::copy`（Ranges 组合风格）

```cpp
#include <algorithm>
#include <ranges>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    std::vector<int> evens;

    auto filtered = v | std::views::filter([](int x) { return x % 2 == 0; });
    std::ranges::copy(filtered, std::back_inserter(evens));

    for (int x : evens) std::cout << x << ' ';
    // 2 4 6 8 10
}
```

这种写法更符合 ranges 的"管道/组合"哲学，尤其当你还需要链式组合其他 view（比如 `transform`、`take` 等）时更自然，例如：

```cpp
// 先筛选偶数，再翻倍，再取前3个，最后复制出来
auto pipeline = v | std::views::filter([](int x) { return x % 2 == 0; })
                  | std::views::transform([](int x) { return x * 2; })
                  | std::views::take(3);

std::vector<int> result;
std::ranges::copy(pipeline, std::back_inserter(result));
// result = {4, 8, 12}
```

## 两者对比

| 方式                             | 优点                             | 适用场景                |
| -------------------------------- | -------------------------------- | ----------------------- |
| `ranges::copy_if`                | 一步到位，直观，无中间视图开销   | 只做"筛选+复制"这一件事 |
| `views::filter` + `ranges::copy` | 可与其他 view 自由组合，惰性求值 | 需要多步链式变换/筛选   |

## 注意事项

无论哪种方式，如果目标容器**没有预先分配足够空间**（比如空的 `vector`），都必须配合插入迭代器（如 `std::back_inserter`），否则会越界写入：

```cpp
// 错误！evens 是空的，没有可写入的空间
std::vector<int> evens;
std::ranges::copy_if(v, evens.begin(), pred); // 未定义行为

// 正确
std::ranges::copy_if(v, std::back_inserter(evens), pred);
```

如果事先知道结果数量（比如先用 `ranges::count_if` 算出满足条件的元素个数），也可以提前 `resize` 目标容器再直接用普通迭代器写入，避免 `back_inserter` 的多次重新分配开销。

## 小结

- **只是复制、不做筛选** → `ranges::copy`
- **复制的同时按条件筛选** → `ranges::copy_if`（简单直接）或 `views::filter` + `ranges::copy`（可组合、可链式扩展）