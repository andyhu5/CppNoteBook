这里说的应该是 **C++20 ranges 库中「view 概念」的一次事后修订（Defect Report，追溯应用到 C++20，但常和 C++23 一起讨论）**，核心内容是：允许 view **拥有（own）** 底层资源，而不再要求"只能是廉价引用包装器"。下面详细解释。

## 1. 最初的 view 概念（C++20 刚发布时）

C++20 ranges 定义 `view` 概念时，要求满足：

- 是一个 `range`
- **拷贝、移动、赋值都必须是 O(1)**（常数时间）
- 语义上"不拥有"底层元素——它只是对某个 range 的"廉价窗口"

这意味着像 `std::vector`、`std::string` 这种"拥有资源、拷贝是 O(n)"的容器，**不满足 view**，只能算 `range`，不能算 `view`。

## 2. 遇到的问题

假设你想把一个**临时对象（右值）**直接接入 range 管道：

```cpp
auto v = std::vector{1, 2, 3, 4, 5} | std::views::filter([](int x){ return x % 2 == 0; });
```

`std::vector{...}` 是右值临时对象。传统 view（比如 `ranges::ref_view`）是靠**引用**包住底层 range 的——但引用绑定到临时对象后，临时对象生命周期一结束就变成悬垂引用（dangling），非常危险。

如果按老规矩，这种写法要么被禁止，要么留下悬垂引用的坑。

## 3. 修订内容（本质就是你说的 DR）

相关提案主要是：

- **P2415R2** *"What is a view?"*
- 配合 **P2325R3** 等一起，对 C++20 的 ranges 规范做了 Defect Report 级别的修正

修订把 view 的要求从：

> 拷贝、移动都必须 O(1)

放宽为：

> **只要求"移动"是 O(1)，拷贝可以是 O(n)，甚至可以直接禁止拷贝（move-only）**

这就是"view with ownership"的含义：**只要移动足够廉价，view 就可以真正拥有底层资源**，不必仅仅是引用包装。

## 4. 由此诞生的 `owning_view`

标准库据此新增了 `std::ranges::owning_view`：

```cpp
namespace std::ranges {
    template<range R>
    class owning_view : public view_interface<owning_view<R>> {
        R r_; // 直接持有对象本体，而不是引用
    public:
        owning_view(owning_view&&) = default;   // 移动 O(1)
        owning_view(const owning_view&) = delete; // 拷贝直接禁止/或 O(n)
        // ...
    };
}
```

当你把**右值**传给 `views::all`（管道运算符背后调用的正是它）时：

- 如果传入的是**左值**（普通变量）→ 用 `ref_view` 包一层引用（老行为，不拥有）
- 如果传入的是**右值**（临时对象）→ 用 `owning_view` **把它移动进来，真正持有所有权**

```cpp
std::vector<int> vec{1,2,3,4,5};

// 左值:被 ref_view 引用包装，不拥有,vec 必须活得比 view 久
auto v1 = std::views::all(vec);

// 右值:被 owning_view "拥有",安全,没有悬垂引用问题
auto v2 = std::views::all(std::vector{1,2,3,4,5});

for (int x : std::vector{10,20,30} | std::views::filter([](int x){ return x > 15; }))
    std::cout << x << ' ';   // 安全:临时 vector 被 owning_view 移动接管
```

## 5. 为什么重要

- 让 range 管道可以**直接、安全地接受临时对象**（右值容器），不用担心悬垂引用
- 扩大了"view"的适用范围：从"只能是廉价引用"扩展到"可以是移动语义下的真正所有者"
- 这类修正虽然是针对 C++20 ranges 规范的缺陷修复（DR），但由于影响面广，很多资料把它和 C++23 ranges 的完善（比如 `views::zip`、`views::chunk` 等新 view 大量依赖这个放宽后的语义）放在一起讨论，所以你会看到 "C++23" 和这个改动经常被提到一起。

**一句话总结**：这次修订把"view = 廉价引用包装器"改成了"view = 移动 O(1) 的 range 包装器（拷贝可贵可禁）"，从而允许 view **真正拥有**底层数据，`owning_view` 就是这一放宽后诞生的具体产物。