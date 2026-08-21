`std::ranges::find_last` 和 `std::ranges::find_last_if`

（连同 `find_last_if_not`）是 C++23 新增的算法，位于 `<algorithm>` 头文件中，是对 C++20 就有的 `ranges::find`/`find_if`/`find_if_not` 的补充——顾名思义，它们查找的是**最后一个**满足条件的元素，而不是第一个。

## 声明（简化版）

```cpp
namespace std::ranges {
    template<forward_range R, class T, class Proj = identity>
        requires indirect_binary_predicate<ranges::equal_to, 
                  projected<iterator_t<R>, Proj>, const T*>
    constexpr borrowed_subrange_t<R> 
        find_last(R&& r, const T& value, Proj proj = {});

    template<forward_range R, class Pred, class Proj = identity>
        requires indirect_unary_predicate<Pred, projected<iterator_t<R>, Proj>>
    constexpr borrowed_subrange_t<R> 
        find_last_if(R&& r, Pred pred, Proj proj = {});

    template<forward_range R, class Pred, class Proj = identity>
        requires indirect_unary_predicate<Pred, projected<iterator_t<R>, Proj>>
    constexpr borrowed_subrange_t<R> 
        find_last_if_not(R&& r, Pred pred, Proj proj = {});
}
```

（也有对应的迭代器/哨兵重载 `(I first, S last, ...)`，与 `find` 类似，这里省略。）

## 功能

- `find_last(r, value)`：从范围中找出**最后一个**等于 `value` 的元素。
- `find_last_if(r, pred)`：找出**最后一个**满足 `pred(elem)` 为真的元素。
- `find_last_if_not(r, pred)`：找出**最后一个**满足 `pred(elem)` 为假的元素。

## 关键区别：返回值是子范围（subrange），不是单个迭代器

这是它们与 `find`/`find_if` 最大的不同点。`ranges::find` 返回一个迭代器，但 `ranges::find_last` 返回的是 `borrowed_subrange_t<R>`，即 **`[找到的位置, 范围末尾)`** 这样一个子范围。

原因很直接：如果只返回一个迭代器指向"最后一个匹配的元素"，那**没有办法同时表示"没找到"**这种情况——因为对 `find`，"没找到"用 `last` 表示；但对 `find_last`，光有一个指向匹配元素的迭代器不足以判断到底是找到了还是没找到（除非再跟 `last` 比较，而这恰好是子范围设计要解决的）。返回子范围后：

- 找到时：`.begin()` 指向匹配的元素，`.end()` 等于原范围的 `end()`
- 没找到时：整个子范围是空的，即 `.begin() == .end() == 原范围的 end()`

这样只需检查返回的子范围是否为空（`empty()`），就能统一判断是否找到。

## 示例

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v = {1, 3, 5, 3, 9, 3, 7};

    // 查找最后一个值为 3 的元素
    auto result = std::ranges::find_last(v, 3);
    if (!result.empty()) {
        std::cout << "找到了，值为: " << *result.begin() << '\n'; // 3
        std::cout << "位置索引: " 
                   << std::distance(v.begin(), result.begin()) << '\n'; // 5
    }

    // 查找最后一个偶数
    auto result2 = std::ranges::find_last_if(v, [](int x) { return x % 2 == 0; });
    std::cout << std::boolalpha << result2.empty() << '\n'; // true，因为没有偶数

    // 查找最后一个不等于 3 的元素
    auto result3 = std::ranges::find_last_if_not(v, [](int x) { return x == 3; });
    std::cout << *result3.begin() << '\n'; // 7（最后一个元素）
}
```

## 支持投影（projection）

和其他 ranges 算法一样，支持 `proj` 参数，方便对结构体成员或经过变换的值做比较：

```cpp
struct Person { std::string name; int age; };

std::vector<Person> people = {
    {"Alice", 30}, {"Bob", 25}, {"Carol", 30}, {"Dave", 22}
};

// 查找最后一个年龄为 30 的人
auto result = std::ranges::find_last(people, 30, &Person::age);
std::cout << result.begin()->name << '\n'; // "Carol"
```

## 为什么不像 `rbegin()/rend()` 反向查找就行了？

确实可以用反向迭代器实现类似效果：

```cpp
auto it = std::ranges::find(v.rbegin(), v.rend(), 3);
// it 是反向迭代器，需要 .base() 转换，还要再 --，容易出错
```

但这种写法：

1. 要求范围本身支持双向迭代器（`bidirectional_range`），而 `find_last` 只要求 `forward_range`，适用范围更广。
2. 反向迭代器转正向迭代器需要额外的 `.base()` 转换并处理偏移问题，容易出 off-by-one 错误。
3. `find_last` 语义更直观，直接表达"找最后一个"的意图，不必绕一层反向迭代器。

## 约束要求

`find_last` 系列只要求 `forward_range`（而不是 `bidirectional_range`），这是因为它们的典型实现方式是**从头正向遍历一遍，记录下最后一次匹配的位置**，并不需要真正反向移动迭代器：

```cpp
// 简化实现思路
auto found = ranges::end(r);
for (auto it = ranges::begin(r); it != ranges::end(r); ++it) {
    if (*it == value) found = it;
}
return {found, ranges::end(r)};
```

（不过对于 `bidirectional_range`/`sized_range`，标准库实现通常会做反向遍历的优化，一旦找到就能立刻返回，不必扫完整个范围。）

## 小结

| 算法               | 查找条件               | 返回类型               |
| ------------------ | ---------------------- | ---------------------- |
| `find`             | 等于 value 的第一个    | 单个迭代器             |
| `find_if`          | 满足 pred 的第一个     | 单个迭代器             |
| `find_last`        | 等于 value 的最后一个  | `subrange`（可能为空） |
| `find_last_if`     | 满足 pred 的最后一个   | `subrange`（可能为空） |
| `find_last_if_not` | 不满足 pred 的最后一个 | `subrange`（可能为空） |

`find_last` 系列是 C++23 对 `find` 家族"反向查找"这一常见需求的正式补齐，通过返回子范围巧妙解决了"如何统一表示找到/未找到"的设计问题。