# C++23：`Compatibility between tuple, pair and tuple-like objects`（P2165）解释

这是 Corentin Jabot 的提案 P2165（标题引用莎士比亚玩了个双关："A tuple by any other name would unpack just as well"），目的是**打通 `std::pair`、`std::tuple`、`std::array` 以及 `ranges::subrange` 等所有满足"tuple 协议"（即支持 `get<N>`、`tuple_size`、`tuple_element`）的类型之间的互操作性**。摘要说得很直白：让 pair、tuple、tuple_cat 以及关联容器都能更好地兼容满足 tuple 协议的对象。

## 1. 解决的问题：这些类型本来"形似而神不通"

C++ 里有好几种"看起来都是固定大小的异构/同构元组"的类型：

```cpp
std::pair<int, double>       p;   // 恰好2个元素
std::tuple<int, double>      t;   // N个元素
std::array<int, 2>           a;   // 同构N个元素
std::ranges::subrange<It>    s;   // 2个元素（begin/end）
```

它们都支持 `std::get<N>`、`std::tuple_size`、`std::tuple_element`，理论上"长得一样"，但在 C++20 里彼此**不能直接构造、赋值、比较**：

```cpp
std::pair<int, double> pp{1, 3.0};
std::tuple<int, double> t{pp};        // C++20之前：OK（tuple有pair构造函数）
std::pair<int, double> pp2{t};        // C++20之前：编译错误！pair 没有从 tuple 构造的能力
```

这就很别扭——`tuple` 能从 `pair` 构造，反过来却不行；`array<int,2>` 想转成 `pair<int,int>` 也没有现成路径。

## 2. P2165 的核心改动：定义 `tuple-like` / `pair-like` concept

标准新增了两个（说明性/exposition-only）concept：

```cpp
// 大致语义：满足 tuple_size / tuple_element / get<N> 协议的类型
template <typename T>
concept tuple-like = /* 满足 tuple_size<T>、tuple_element、get<N>(t) 等要求 */;

template <typename T>
concept pair-like = tuple-like<T> && tuple_size_v<remove_cvref_t<T>> == 2;
```

有了这套 concept 之后，标准库大量原本"只认 `pair`/`tuple` 自己"的接口，被放宽成"只要满足 `tuple-like`/`pair-like` 就行"：

### `pair` 现在可以从任意 pair-like 构造/赋值

```cpp
std::array<int, 2> arr = {1, 2};
std::pair<int, int> pp(arr);          // C++23: OK！之前不行
pp = arr;                              // 赋值也支持

std::tuple<int, double> t{1, 3.0};
std::pair<int, double> pp2(t);        // C++23: OK
```

### `tuple_cat`、`apply`、`make_from_tuple` 都放宽为接受任意 tuple-like

```cpp
std::array<int, 2> a = {1, 2};
std::tuple<double> t2{3.0};

// C++23之前 tuple_cat 只认 tuple（某些实现魔改支持 pair/array，但不是标准保证）
// C++23: 标准明确要求支持任意 tuple-like
auto result = std::tuple_cat(a, t2);   // OK: tuple<int, int, double>

std::pair<int, int> pp = {1, 2};
std::apply([](int a, int b) { return a + b; }, pp);  // 早已支持pair，现在泛化到所有tuple-like
```

### 关联容器构造函数支持 tuple-like

```cpp
std::array<int, 2> key_value = {1, 100};
std::map<int, int> m;
m.emplace(key_value);   // C++23: 更顺畅地把 tuple-like 拆成 key/value
```

## 3. 为什么这对 Ranges 生态特别重要：`zip` 与 `sortable`

这一点在提案讨论和后续 LWG issue（issue 3865）里被专门提到：`views::zip` 产生的 range，其 `reference` 类型是 `pair<int&, int&>`，`value_type` 是 `pair<int, int>`。

在 C++20 里，`std::pair` **没有异构比较**（heterogeneous comparison）能力，也就是说 `pair<int&, int&>` 和 `pair<int, int>` 之间不能直接比较大小——而这恰恰是 `std::ranges::sortable` concept 所要求的（排序算法内部经常需要比较"引用类型"和"值类型"）。这导致：

```cpp
std::vector<int> a = {3, 1, 2};
std::vector<int> b = {30, 10, 20};

auto zipped = std::views::zip(a, b);
std::ranges::sort(zipped);   // C++20 之前：无法满足 sortable，编译失败
```

P2165 给 `pair`/`tuple` 之间打通了异构构造、赋值、`tuple_cat`（比较部分后来因为 `common_reference` 特化不完整，在 R4 里被收窄移除，留给后续单独解决），使得类似 `zip` 这种"pair of references"和"pair of values"混用的场景变得可用——这段现在是合法 C++23 代码，在 P2165 之前是不行的。

## 4. R4 版本收窄了什么（值得注意的边界）

从 P2165R4 的修订记录可以看到，最终版本相比早期版本做了收缩：

- **移除了 `pair` 和 tuple-like 之间的比较运算符**——因为要满足 `equality_comparable` 需要有对应的 `common_reference` 特化，而这块当时还没做完整，所以先砍掉，避免引入语义不完整的比较行为
- **移除了对 `use_allocator` 构造的支持**——LWG 讨论后认为这块复杂度和收益不成正比，先不做

这说明这是一个**渐进式**的兼容性改进：先把构造、赋值、`tuple_cat`/`apply` 这些"结构性"操作打通，比较运算符这类需要更完整基础设施（`common_reference`）支持的部分留给后续提案继续完善。

## 5. 一句话总结

**P2165 让 `std::pair`、`std::tuple`、`std::array`、`ranges::subrange` 等一切满足"tuple 协议"（`tuple_size`/`tuple_element`/`get<N>`）的类型能够互相构造、赋值，并让 `tuple_cat`、`apply`、`make_from_tuple` 和关联容器都统一接受任意 tuple-like 类型**——本质上是把"tuple 协议"从"仅仅是解构语法糖"提升为一套真正可以在标准库各处互操作的通用接口，也是让 `views::zip` 之类的现代 Ranges 特性能顺利工作的重要基础设施。