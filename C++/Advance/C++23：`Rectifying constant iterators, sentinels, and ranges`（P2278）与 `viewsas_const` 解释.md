# C++23：`Rectifying constant iterators, sentinels, and ranges`（P2278）与 `views::as_const` 解释

这个提案（P2278R4）解决的是 Ranges 库里一个存在已久、比较隐蔽的 **const 正确性 bug**：`ranges::cbegin` 对某些 view（比如 `std::span`）**并不能保证返回真正的常量迭代器**。cppreference 的总结是：ranges::cbegin 和其他类似的返回常量迭代器的工具，应该对"浅 const"的 view（比如 std::span）也完全保证这一点。

## 1. 问题根源：什么是"浅 const"（shallow-const）view

`std::span` 这类 view 有个特点：它自己是"引用语义"的轻量包装器，`const span<int>` 只是让 `span` 对象本身（内部的指针和长度）不能改，但**并不会**让它指向的元素变成 const：

```cpp
void foo(const std::span<int> sp) {
    sp[0] = 42;  // 合法！span 是 const，但元素不是 const
}
```

这就叫"浅 const"（shallow const）：外层加 `const` 没有传导到元素上。这跟 `vector` 完全不同——`const vector<int>` 里的元素**确实**变成 const 了。

## 2. Bug 具体表现在哪

在 P2278 之前，`ranges::cbegin(r)` 的实现大致是：**如果 `r` 是左值，就先把它转成 `const`，再调用 `begin`**。逻辑是"给我一个 const 迭代器，那就先把 range 变成 const 再取 begin 呗"。

对 `vector` 这种"深 const"容器没问题：

```cpp
std::vector<int> v = {1, 2, 3};
auto it = std::ranges::cbegin(v);
*it;      // int const&，正确，不能改
```

但对 `span` 这种浅 const view，`const span<int>` 的 `begin()` 返回的迭代器解引用出来还是 `int&`，**不是** `int const&`：

```cpp
std::span<int> sp = v;
auto it = std::ranges::cbegin(sp);
*it = 100;   // C++20：这竟然是合法的！违背了 "cbegin 应该给常量迭代器" 的直觉
```

这明显违反了 `cbegin` 名字所暗示的承诺——"c" 代表 const，用户期望通过 `cbegin` 拿到的迭代器**绝对不能**用来修改元素。

## 3. P2278 的修复方式

P2278 引入了 `std::basic_const_iterator`（`const_iterator` 的一个包装适配器），如果检测到某个 range 的普通迭代器解引用出来不是 const 引用，就**用这个包装器强行包一层**，让它的 `operator*` 返回 const 引用：

```cpp
// 概念示意：const_iterator<I> 包装迭代器 I
template <input_iterator I>
class const_iterator {
    I it;
public:
    decltype(auto) operator*() const {
        return static_cast<const iter_reference_t<I>>(*it);  // 强制转成 const 引用
    }
    // ...
};
```

修复之后：

```cpp
std::span<int> sp = v;
auto it = std::ranges::cbegin(sp);
*it = 100;   // C++23：编译错误！现在真的是 const 了
```

同时增加了对应的 concept/trait：`std::const_iterator`、`std::const_sentinel`，以及 `std::ranges::const_iterator_t`、`std::ranges::const_sentinel_t` 这类别名，方便在模板代码里推导"常量版本的迭代器类型"应该是什么。

## 4. `std::views::as_const`

配合这个修复，P2278 同时提供了一个新的 range adaptor：`views::as_const`，作用是把任意 range **转换成一个元素为 const 的视图**，无论底层 range 是不是浅 const 的：

```cpp
#include <ranges>
#include <span>
#include <vector>

std::vector<int> v = {1, 2, 3};
std::span<int> sp = v;

for (auto& x : sp | std::views::as_const) {
    // x 的类型是 const int&，无法修改
    std::print("{}\n", x);
}

// 等价于直接用 ranges::cbegin/cend 得到的效果，但可以像管道一样组合
auto cv = v | std::views::as_const;
```

它的行为：

- 如果传入的 range 本身元素已经是 const 的（比如它是 `const vector<int>&`），`as_const` 会尽量"什么都不做"直接透传，避免不必要的包装开销
- 如果元素不是 const 的（比如 `span<int>`），就用 `const_iterator` 包装器包一层，强制变成常量视图
- 可以自然地和其他 view adaptor 用 `|` 管道组合：

```cpp
auto result = v 
    | std::views::filter([](int x) { return x > 1; })
    | std::views::as_const;   // 过滤之后再"上锁"成只读视图，防止后续代码误改
```

## 5. 为什么这个改动很重要

这不只是个边角案例修复，而是关系到 **API 契约能不能被信任**：

- 如果你写一个函数接受 `range auto&& r`，然后用 `ranges::cbegin(r)` 来遍历并"承诺"不修改元素，在 C++20 里这个承诺对 `span` 之类的 view 是**不成立的**——这是个真实的 const 正确性漏洞，可能导致意料之外的数据被修改
- `views::as_const` 给了用户一个显式、通用的手段，把任何 range（不管深浅 const）转成"只读视图"，这在写通用泛型代码、或者想暴露一个只读接口给调用者时非常有用

```cpp
class MyContainer {
    std::vector<int> data_;
public:
    // 对外只暴露只读视图，即便内部用的是可变的 span 之类
    auto view() const {
        return std::span(data_) | std::views::as_const;
    }
};
```

## 6. 一句话总结

**P2278 修复了 `cbegin`/`cend` 对"浅 const"view（如 `span`）失效的 const 正确性 bug，引入 `const_iterator`/`const_sentinel` 包装器保证"c 前缀"名副其实；`views::as_const` 则是配套提供的显式 adaptor，可以把任意 range 转成保证只读的常量视图**，两者共同让 Ranges 库里"const 到底传没传导到元素"这件事变得可预测、可信赖。