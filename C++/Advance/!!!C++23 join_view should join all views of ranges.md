## 

`views::join` 在 C++23 里得到了两处相互独立的改进：P2328 修复了 `join_view` 本身的一个约束缺陷；P2441 新增了 `views::join_with` 作为 `join` 的带分隔符版本。两者都和"join 应该能处理所有 range of ranges"这个目标有关，但解决的是不同层面的问题。

## P2328：join_view should join all views of ranges

**C++20 的限制。**`join_view` 在 C++20 里要求内层 range 必须是 view（满足 `std::ranges::view` concept）或者左值引用。这意味着当外层 range 的 `operator*` 返回一个按值产生的 range（prvalue，如 `std::vector<int>`）时，`join` 拒绝编译。这在 `transform | join` 这个极为常见的组合中频繁触发：

```cpp
auto make_inner(int n) -> std::vector<int> { return {n, n * 10}; }

auto outer = std::vector<int>{1, 2, 3};

// C++20 中 transform 的每个元素是 prvalue vector<int>，
// 它不是 view，join_view 的约束会直接拒绝这个输入。
auto r = outer | std::views::transform(make_inner) | std::views::join;  // C++20: 编译失败
```

**根本原因。** prvalue range 的生命周期问题让 `join_view` 的迭代器设计变得棘手。当内层迭代器指向某个 prvalue range 的内部时，需要有人持有那个 range——否则它在表达式结束后就被销毁了，迭代器悬空。C++20 的做法是干脆禁止这种情况，而不是处理它。

**C++23 的修复。** P2328 在 `join_view` 内部引入了一个 `non-propagating-cache<InnerRange>`（本质上是一个不传播拷贝/移动语义的 `optional`），专门用来持有当前正在迭代的那个 prvalue 内层 range。"不传播"意味着 copy 或 move `join_view` 本身时，这个缓存会被重置为空，而不是把内层 range 也一并复制过去——这正是正确的语义，因为那个缓存对象是迭代状态，不应该随着 view 的复制而克隆。

```cpp
// C++23 中可以直接 join prvalue range
auto r = outer | std::views::transform(make_inner) | std::views::join;

for (int x : r)
    std::cout << x << ' ';   // 1 10 2 20 3 30
```

迭代器语义上的影响：当内层 range 是 prvalue 时，产生的迭代器类别降级为 input iterator（因为缓存中的 range 只能被迭代一次），但这通常正是实际代码所需要的类别。

## P2441：views::join_with

C++20 的 `views::join` 把 range of ranges 展平成一个 range，但没有任何方式在相邻的内层 range 之间插入分隔符。这对字符串拼接、CSV 构建等场景来说是一个缺口。`views::join_with` 补上了这个缺口，其语义正是 `views::split` 的逆操作：

```cpp
// 单字符分隔符
std::vector<std::string_view> words = {"Hello", "world", "C++23"};
auto sentence = words | std::views::join_with(' ');
// 产生：H e l l o   w o r l d   C + + 2 3

// range 分隔符
auto delimited = words | std::views::join_with(", "sv);
// 产生：H e l l o ,   w o r l d ,   C + + 2 3
```

注意结果是 range of `char`，不是 range of `string`，因为 `join_with` 把内层 range（每个 `string_view`）和分隔符全部展平到同一层。如果需要最终的字符串，要配合 `ranges::to<std::string>()`（也是 C++23 新增的）：

```cpp
std::string result = words | std::views::join_with(", "sv) | std::ranges::to<std::string>();
// result == "Hello, world, C++23"
```

提案明确描述了 `split` 和 `join_with` 的对称性：

```cpp
std::string_view csv = "a,b,c";
auto parts   = csv    | std::views::split(',');
auto rebuilt = parts  | std::views::join_with(',');
// rebuilt 的内容与 csv 完全相同
```

**分隔符类型的约束。** 分隔符可以是单个元素值，也可以是一个 range。当分隔符是单个值时，`join_with` 内部把它包进 `views::single` 来统一处理。内层 range 的 value type 和分隔符的 value type 必须有公共类型（`std::common_type`），否则产生的 range 的 value type 无法确定。

## 迭代器类别的变化规则

两个 view 的迭代器类别都依赖输入：

```
join_view 产生 forward_range 的条件：
  外层 range 是 forward，且内层 range 是 lvalue reference 或 view
  （prvalue 内层 range → 只能是 input_range）

join_with_view 产生 forward_range 的条件：
  外层 range 和分隔符 range 都是 forward，且内层 range 是 lvalue reference 或 view
```

实际使用中，处理 `vector<vector<int>>` 这类内层 range 是左值引用的情况，两者都能提供 forward iterator；处理 `transform` 产生 prvalue 的情况，则只能得到 input iterator，但这已经足够覆盖绝大多数展平和字符串拼接的用例。