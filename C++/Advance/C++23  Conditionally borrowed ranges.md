**C++ DR20: Conditionally borrowed ranges** 是 C++23 的一个 **Defect Report（缺陷修复）**，它解决的是 **Ranges 生命周期（lifetime）** 的一个设计问题。

一句话概括：

> **让某些 View 是否属于 `borrowed_range`，根据底层 Range 是否安全来决定，而不是一刀切。**

这是 C++20 Ranges 中比较重要的一项修复。

------

## 1. 什么是 borrowed_range？

首先理解 `borrowed_range`。

考虑下面的代码：

```cpp
std::vector<int> v{1,2,3};

auto it = std::ranges::find(v, 2);
```

这里没有问题。

因为

```
v
│
├── iterator
└── 数据仍然存在
```

Iterator 指向的数据不会消失。

但是：

```cpp
auto it =
    std::ranges::find(std::vector{1,2,3}, 2);
```

这里就危险了。

临时对象：

```
find( std::vector{...}, 2 )

        │
        ▼
    vector 已析构

iterator
    │
    ▼
悬空(dangling)
```

因此 C++20 引入了

```cpp
std::ranges::borrowed_range
```

它表示：

> **即使 Range 本身销毁，Iterator 仍然有效。**

例如：

```cpp
std::span
```

就是 borrowed_range。

因为：

```cpp
std::span<int> s(arr);
```

span 本身只是：

```
pointer
length
```

即使 span 对象销毁，

```
数组
```

还在。

所以 iterator 没问题。

------

## 2. 哪些不是 borrowed_range？

例如：

```cpp
std::vector
```

不是。

因为：

```
vector
│
├── buffer
└── iterator
```

vector 析构：

```
buffer
↓

释放
```

iterator 全部失效。

------

## 3. C++20 的问题

很多 View：

例如

```cpp
views::take
views::drop
views::reverse
views::filter
```

其实只是包装了一个 Range。

例如：

```cpp
auto r =
    vec | std::views::take(5);
```

实际上：

```
take_view
    │
    ▼
vector
```

如果：

```
vector

是 borrowed
```

那么：

```
take_view

其实也应该 borrowed
```

反过来：

如果：

```
vector

不是 borrowed
```

那么：

```
take_view

也不能 borrowed
```

------

但是 C++20 并没有这样设计。

很多 View：

```
永远 borrowed
```

或者：

```
永远不是 borrowed
```

都不合理。

------

## 举例

例如：

```cpp
std::span<int> s(arr);

auto t =
    s | std::views::take(3);
```

底层：

```
span
```

本来就是 borrowed。

但是：

```
take_view<span>
```

在 C++20 中，

很多实现：

```
不是 borrowed
```

于是：

```cpp
auto it =
    std::ranges::begin(
        s | std::views::take(3));
```

返回：

```
dangling
```

实际上完全没有必要。

------

反过来：

```
vector
↓

take_view
```

又可能错误地允许 iterator 活下来。

生命周期判断混乱。

------

## 4. DR20 如何修复？

修复思想非常简单：

> **View 是否 borrowed，取决于它包装的底层 Range。**

例如：

```cpp
template<class V>
class take_view;
```

以前：

```
take_view

↓

固定 borrowed
```

或者：

```
固定不是 borrowed
```

现在：

```
take_view<V>

↓

borrowed_range<V>

?

borrowed

:

不是 borrowed
```

也就是：

```
底层 borrowed

↓

View borrowed
底层不是 borrowed

↓

View 也不是
```

因此叫：

> **Conditionally borrowed ranges**

即：

**有条件地成为 borrowed_range。**

------

## 哪些 View 受影响？

很多标准 View 都改成了这种规则，例如：

- `take_view`
- `drop_view`
- `reverse_view`
- `common_view`
- `elements_view`
- `keys_view`
- `values_view`
- `adjacent_view`
- `slide_view`
- `chunk_view`
- `enumerate_view`（后续标准）

几乎所有只包装底层 Range 的 View 都采用这一策略。

------

## 举例

### 情况一：span

```cpp
std::span<int> s(arr);

auto v =
    s | std::views::take(3);
```

现在：

```
span

borrowed
        │
        ▼
take_view

borrowed
```

因此：

```cpp
auto it =
    std::ranges::begin(v);
```

完全安全。

------

### 情况二：临时 vector

```cpp
auto v =
    std::vector{1,2,3}
    | std::views::take(2);
```

这里：

```
vector

不是 borrowed
```

因此：

```
take_view

↓

也不是 borrowed
```

如果：

```cpp
auto it =
    std::ranges::begin(v);
```

标准会正确返回：

```cpp
std::ranges::dangling
```

而不是一个危险 iterator。

------

## 总结

DR20 **Conditionally borrowed ranges** 并没有引入新的 View 或新的算法，而是修正了 C++20 对 `borrowed_range` 的传播规则。

核心思想是：

- `borrowed_range` 表示 **Range 销毁后，其迭代器仍然有效**。
- C++20 中许多 View 对 `borrowed_range` 的支持是固定的，无法正确反映底层 Range 的生命周期特性。
- C++23 改为 **条件传播（conditional propagation）**：如果底层 Range 是 `borrowed_range`，包装它的 View 也自动是 `borrowed_range`；否则也不是。

这一修复让 Ranges 的生命周期推导更加准确，减少了不必要的 `std::ranges::dangling`，同时也避免了错误地返回悬空迭代器，是 C++23 对 Ranges 生命周期模型的一项重要完善。**C++ DR20: Conditionally borrowed ranges** 是 C++23 的一个 **Defect Report（缺陷修复）**，它解决的是 **Ranges 生命周期（lifetime）** 的一个设计问题。

一句话概括：

> **让某些 View 是否属于 `borrowed_range`，根据底层 Range 是否安全来决定，而不是一刀切。**

这是 C++20 Ranges 中比较重要的一项修复。

------

## 1. 什么是 borrowed_range？

首先理解 `borrowed_range`。

考虑下面的代码：

```cpp
std::vector<int> v{1,2,3};

auto it = std::ranges::find(v, 2);
```

这里没有问题。

因为

```
v
│
├── iterator
└── 数据仍然存在
```

Iterator 指向的数据不会消失。

但是：

```cpp
auto it =
    std::ranges::find(std::vector{1,2,3}, 2);
```

这里就危险了。

临时对象：

```
find( std::vector{...}, 2 )

        │
        ▼
    vector 已析构

iterator
    │
    ▼
悬空(dangling)
```

因此 C++20 引入了

```cpp
std::ranges::borrowed_range
```

它表示：

> **即使 Range 本身销毁，Iterator 仍然有效。**

例如：

```cpp
std::span
```

就是 borrowed_range。

因为：

```cpp
std::span<int> s(arr);
```

span 本身只是：

```
pointer
length
```

即使 span 对象销毁，

```
数组
```

还在。

所以 iterator 没问题。

------

## 2. 哪些不是 borrowed_range？

例如：

```cpp
std::vector
```

不是。

因为：

```
vector
│
├── buffer
└── iterator
```

vector 析构：

```
buffer
↓

释放
```

iterator 全部失效。

------

## 3. C++20 的问题

很多 View：

例如

```cpp
views::take
views::drop
views::reverse
views::filter
```

其实只是包装了一个 Range。

例如：

```cpp
auto r =
    vec | std::views::take(5);
```

实际上：

```
take_view
    │
    ▼
vector
```

如果：

```
vector

是 borrowed
```

那么：

```
take_view

其实也应该 borrowed
```

反过来：

如果：

```
vector

不是 borrowed
```

那么：

```
take_view

也不能 borrowed
```

------

但是 C++20 并没有这样设计。

很多 View：

```
永远 borrowed
```

或者：

```
永远不是 borrowed
```

都不合理。

------

## 举例

例如：

```cpp
std::span<int> s(arr);

auto t =
    s | std::views::take(3);
```

底层：

```
span
```

本来就是 borrowed。

但是：

```
take_view<span>
```

在 C++20 中，

很多实现：

```
不是 borrowed
```

于是：

```cpp
auto it =
    std::ranges::begin(
        s | std::views::take(3));
```

返回：

```
dangling
```

实际上完全没有必要。

------

反过来：

```
vector
↓

take_view
```

又可能错误地允许 iterator 活下来。

生命周期判断混乱。

------

## 4. DR20 如何修复？

修复思想非常简单：

> **View 是否 borrowed，取决于它包装的底层 Range。**

例如：

```cpp
template<class V>
class take_view;
```

以前：

```
take_view

↓

固定 borrowed
```

或者：

```
固定不是 borrowed
```

现在：

```
take_view<V>

↓

borrowed_range<V>

?

borrowed

:

不是 borrowed
```

也就是：

```
底层 borrowed

↓

View borrowed
底层不是 borrowed

↓

View 也不是
```

因此叫：

> **Conditionally borrowed ranges**

即：

**有条件地成为 borrowed_range。**

------

## 哪些 View 受影响？

很多标准 View 都改成了这种规则，例如：

- `take_view`
- `drop_view`
- `reverse_view`
- `common_view`
- `elements_view`
- `keys_view`
- `values_view`
- `adjacent_view`
- `slide_view`
- `chunk_view`
- `enumerate_view`（后续标准）

几乎所有只包装底层 Range 的 View 都采用这一策略。

------

## 举例

### 情况一：span

```cpp
std::span<int> s(arr);

auto v =
    s | std::views::take(3);
```

现在：

```
span

borrowed
        │
        ▼
take_view

borrowed
```

因此：

```cpp
auto it =
    std::ranges::begin(v);
```

完全安全。

------

### 情况二：临时 vector

```cpp
auto v =
    std::vector{1,2,3}
    | std::views::take(2);
```

这里：

```
vector

不是 borrowed
```

因此：

```
take_view

↓

也不是 borrowed
```

如果：

```cpp
auto it =
    std::ranges::begin(v);
```

标准会正确返回：

```cpp
std::ranges::dangling
```

而不是一个危险 iterator。

------

## 总结

DR20 **Conditionally borrowed ranges** 并没有引入新的 View 或新的算法，而是修正了 C++20 对 `borrowed_range` 的传播规则。

核心思想是：

- `borrowed_range` 表示 **Range 销毁后，其迭代器仍然有效**。
- C++20 中许多 View 对 `borrowed_range` 的支持是固定的，无法正确反映底层 Range 的生命周期特性。
- C++23 改为 **条件传播（conditional propagation）**：如果底层 Range 是 `borrowed_range`，包装它的 View 也自动是 `borrowed_range`；否则也不是。

这一修复让 Ranges 的生命周期推导更加准确，减少了不必要的 `std::ranges::dangling`，同时也避免了错误地返回悬空迭代器，是 C++23 对 Ranges 生命周期模型的一项重要完善。