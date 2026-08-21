`std::views::chunk` 和 `std::views::chunk_by` 是 **C++23** 新增的两个 Range Adapter，定义在 `<ranges>` 中，来源于提案 **P2442R1**。它们的共同目标是**把一个 Range 拆分成多个子 Range（Chunk）**，但拆分规则不同：

- **`views::chunk(N)`**：按固定大小分组。
- **`views::chunk_by(pred)`**：按相邻元素关系分组。

虽然名字相似，但用途完全不同。

------

# 1. `std::views::chunk`

它表示：

> **每 N 个元素组成一个子 Range。**

例如：

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main()
{
    std::vector<int> v{1,2,3,4,5,6,7};

    for (auto group : v | std::views::chunk(3))
    {
        for (int x : group)
            std::cout << x << ' ';
        std::cout << '\n';
    }
}
```

输出：

```text
1 2 3
4 5 6
7
```

可以理解为：

```text
原序列

1 2 3 4 5 6 7

↓

chunk(3)

[1 2 3]
[4 5 6]
[7]
```

最后一个 Chunk 可以不足 3 个元素。

------

## 返回类型

注意：

```cpp
auto group
```

不是：

```cpp
std::vector<int>
```

而是：

```text
一个 View（subrange）
```

所以：

```cpp
group.begin();
group.end();
```

都可以继续使用。

------

## 配合 Pipeline

例如：

```cpp
auto r =
    vec
    | std::views::chunk(4)
    | std::views::transform([](auto g)
      {
          return std::ranges::distance(g);
      });
```

输出：

```text
4
4
2
```

表示每个 Chunk 的大小。

------

# 2. `std::views::chunk_by`

```c

#include <ranges>
#include <vector>
#include <print>
int main() {
  std::vector<int> v{1, 2, 3, 10, 11, 20, 30, 31};

  auto groups =
      v | std::views::chunk_by([](int a, int b) { return b == a + 1; });

    for (auto&& group : groups) {
        std::println("Group:");
        for (auto&& item : group) {
            std::println("  {}", item);
        }
    }

    std::vector<int> v2{1, 1, 1, 2, 2, 3, 4, 4};

    // auto r = v2 | std::views::chunk_by([](int a, int b) { return a == b; });
    auto r = v2 | std::views::chunk_by(std::equal_to{});


    for (auto&& group : r) {
        std::println("Group:");
        for (auto&& item : group) {
            std::println("  {}", item);
        }
    }

    std::vector<std::string> words{"apple", "ant", "banana", "boat", "cat"};

    auto r2 = words | std::views::chunk_by([](const auto &a, const auto &b) {
               return a[0] == b[0];
             });
    for(auto&& group : r2) {
        std::println("Group:");
        for (auto&& item : group) {
            std::println("  {}", item);
        }
    }
}

```

它表示：

> **根据相邻两个元素是否满足谓词决定是否属于同一个 Chunk。**

语法：

```cpp
views::chunk_by(pred)
```

其中：

```cpp
pred(a, b)
```

判断：

> 当前元素和下一个元素是否仍属于同一组。

------

## 最简单例子

```cpp
std::vector<int> v{
    1,2,3,
    10,11,
    20,
    30,31
};

auto groups =
    v
    | std::views::chunk_by(
        [](int a, int b)
        {
            return b == a + 1;
        });
```

输出：

```text
1 2 3
10 11
20
30 31
```

过程：

```text
1 2

连续

继续

2 3

连续

继续

3 10

不连续

切开

10 11

连续

继续

11 20

切开
```

------

# 再举一个例子

按照是否相同分组：

```cpp
std::vector<int> v{
    1,1,1,
    2,2,
    3,
    4,4
};

auto r =
    v
    | std::views::chunk_by(
        std::equal_to{}
      );
```

得到：

```text
1 1 1
2 2
3
4 4
```

这类似于：

```text
Run-Length Encoding
```

第一步。

------

# 按字符串首字母分组

例如：

```cpp
std::vector<std::string> words{
    "apple",
    "ant",
    "banana",
    "boat",
    "cat"
};
```

可以：

```cpp
auto r =
words
| std::views::chunk_by(
[](const auto& a,
   const auto& b)
{
    return a[0]==b[0];
});
```

得到：

```text
apple
ant

banana
boat

cat
```

注意：

这里只比较：

```text
相邻元素
```

不会排序。

------

# `chunk` 与 `chunk_by` 的区别

假设：

```text
1 2 3 10 11 20 30 31
```

------

### chunk(3)

固定数量：

```text
[1 2 3]
[10 11 20]
[30 31]
```

与元素值无关。

------

### chunk_by

连续关系：

```text
[1 2 3]
[10 11]
[20]
[30 31]
```

与元素个数无关。

------

# 为什么要增加 `chunk_by`？

以前只能自己写：

```cpp
for(...)
{
    if(...)
        ...
}
```

或者：

```cpp
find_if
adjacent_find
```

逻辑复杂。

现在：

```cpp
views::chunk_by(...)
```

直接表达：

> 根据相邻元素关系分组。

代码可读性高很多。

------

# 返回的是什么？

例如：

```cpp
auto groups =
v
| views::chunk(3);
```

类型：

```text
chunk_view<vector<int>>
```

每次：

```cpp
for(auto group : groups)
```

得到：

```text
subrange
```

仍然是一个 Range。

所以：

```cpp
for(auto x : group)
```

继续遍历。

------

# 与 `slide` 的区别

例如：

```text
1 2 3 4 5
```

## chunk(3)

```text
[1 2 3]
[4 5]
```

互不重叠。

------

## slide(3)

```text
[1 2 3]
[2 3 4]
[3 4 5]
```

滑动窗口。

这是 C++23 另一个 View。

------

# 与 `stride` 的区别

例如：

```text
1 2 3 4 5 6
views::stride(2)
```

得到：

```text
1 3 5
```

不是分组。

------

# 总结

| View                    | 分组依据           | 示例                   |
| ----------------------- | ------------------ | ---------------------- |
| `views::chunk(n)`       | 固定元素个数       | `[1 2 3] [4 5 6]`      |
| `views::chunk_by(pred)` | 相邻元素满足谓词   | `[1 2 3] [10 11] [20]` |
| `views::slide(n)`       | 固定窗口，允许重叠 | `[1 2 3] [2 3 4]`      |
| `views::stride(n)`      | 固定步长跳过元素   | `1 3 5`                |

**一句话概括：**

- **`std::views::chunk(n)`**：按固定大小切分 Range，每个子 Range 最多包含 `n` 个元素。
- **`std::views::chunk_by(pred)`**：按相邻元素之间的关系切分 Range，只要相邻两个元素满足谓词，就继续归入同一个子 Range；一旦不满足，就开始新的分组。

这两个适配器都是**惰性（lazy）**的，不会复制元素，仅在遍历时动态生成各个子 Range，因此非常适合与其他 Ranges 管道组合使用。