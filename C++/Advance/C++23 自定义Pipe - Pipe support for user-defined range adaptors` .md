# C++23：`P2387R3` —— `Pipe support for user-defined range adaptors` 解释

这是 Barry Revzin 提出的提案 P2387，标题是"为用户自定义 range adaptor 提供管道支持"。它解决的问题是：**C++20 的 Ranges 库里，`|` 管道语法是标准库自己"私有"的黑魔法，普通用户没办法给自己写的 range adaptor 加上同样的 `|` 支持**。C++23 把这套机制标准化、暴露出来了。

## 1. 背景问题：管道语法为什么只有标准库能用

C++20 里我们已经很熟悉这种写法：

```cpp
auto result = data 
            | std::views::filter(pred) 
            | std::views::transform(func);
```

`std::views::filter(pred)` 返回的其实是一个"待定"的 adaptor closure 对象，`|` 运算符把它和左边的 range 组合起来，产生新的 view。

但如果你自己写了一个 range adaptor（比如实现一个 `my_views::chunk` 或者某个自定义的过滤/变换操作），在 C++20 里**没有任何标准化、简便的方式**让它也享受同样的 `|` 语法支持。你要么：

- 手动为你的每个 adaptor 重载 `operator|`，自己处理"部分应用"（先绑定参数，等 range 来了再真正调用）的逻辑，容易写错、代码重复
- 参考 range-v3 库里那套非标准、比较复杂的内部机制去山寨一份

## 2. P2387 的核心贡献：`range_adaptor_closure`

标准新增了一个基类模板 `std::ranges::range_adaptor_closure`（还有相关的 concept `std::ranges::range_adaptor_closure_object`），只要你自己的 adaptor closure 类型继承自它，就自动获得完整的管道支持，包括：

- 和 range 的 `|` 组合（`range | closure`）
- 多个 closure 之间的组合（`closure1 | closure2` 产生一个新的组合 closure）
- 正确的值类别（左值/右值）转发

```cpp
#include <ranges>

// 假设你想写一个"取偶数索引元素"的 adaptor
struct even_indices_fn : std::ranges::range_adaptor_closure<even_indices_fn> {
    template <std::ranges::viewable_range R>
    constexpr auto operator()(R&& r) const {
        return std::views::filter(r, [i = 0](auto&&) mutable { return i++ % 2 == 0; });
        // (示意实现)
    }
};

inline constexpr even_indices_fn even_indices;

// 现在可以直接用管道语法了！
auto result = data | even_indices;                 // OK
auto piped  = even_indices | std::views::transform(f);  // adaptor之间也能组合
```

在这之前，如果你想让 `even_indices_fn` 支持 `|`，需要自己手写：

```cpp
// C++20：需要自己重载 operator|，还要小心处理各种值类别和组合情况
template <std::ranges::viewable_range R>
constexpr auto operator|(R&& r, even_indices_fn f) {
    return f(std::forward<R>(r));
}
// 如果还想支持和别的 adaptor 组合，逻辑会更复杂……
```

## 3. 为什么这个特性属于"基础设施"而不是"面向用户的 view"

值得注意的是，P2387 本身**不新增任何具体的 view**（比如它不像 `views::chunk`、`views::zip` 那样直接提供一个新的过滤/变换操作），它提供的是一套**给所有人（包括标准库自己和第三方库）复用的通用管道机制**。事实上，C++23 标准库内部很多新的 view adaptor（比如 `views::chunk_by`、`views::adjacent` 等）本身也是基于这套 `range_adaptor_closure` 机制实现的，等于是"标准库先给自己用的私有能力，现在开放给所有用户平等使用"。

也正因为这个定位，它被 Ranges 委员会总结为 C++23 对 ranges 的"通用性增强"之一，和 `ranges::to`（收集到容器）、`Formatting Ranges`（格式化）并列，都属于"提升 Ranges 整体易用性的基础设施"，而不是某个具体的新算法或新 view。

## 4. 实际效果

有了这个特性之后，写第三方 range 处理库（或者你自己项目里的工具库）时，可以做到跟标准库 `views::` 系列**风格完全一致、体验无差别**：

```cpp
namespace my_lib::views {
    struct chunk_fn : std::ranges::range_adaptor_closure<chunk_fn> {
        std::size_t n;
        constexpr auto operator()(std::ranges::viewable_range auto&& r) const {
            // ... 实现分块逻辑
        }
    };

    inline constexpr auto chunk(std::size_t n) {
        return chunk_fn{n};   // 返回一个可以直接用 | 组合的 closure
    }
}

auto result = data 
            | my_lib::views::chunk(3) 
            | std::views::transform(sum);   // 自定义 adaptor 和标准库 adaptor 无缝混用
```

## 5. 一句话总结

**P2387 把"range adaptor 如何支持 `|` 管道语法"这套原本只有标准库能用的私有机制，通过 `std::ranges::range_adaptor_closure` 标准化并开放给所有用户，让第三方或自定义的 range adaptor 可以像标准库的 `views::` 系列一样自然地被组合、管道化使用**，这是 C++23 Ranges 生态里非常重要的一块"可扩展性"基础设施，也是很多 C++23 新增 view（以及未来第三方库）能够以一致风格实现的底层支撑。