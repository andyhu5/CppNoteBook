**C++23 DR20（对应 P2325R3）的主要变化是：`std::ranges::view` 概念不再要求 `default_initializable`（默认可初始化）。**

### 1. 背景：C++20 原本的问题

在 C++20 中，`view` 概念的定义是：

```cpp
template <class T>
concept view = range<T> && movable<T> && default_initializable<T> && enable_view<T>;
```

- `default_initializable` 要求类型能默认构造（`T t{};` 或 `T()` 合法）。
- 这个要求**很多时候没有实际意义**，却强迫许多 view 必须提供一个“有意义”的默认状态。
- 结果导致一些合理的 view（如持有不可默认构造的成员、lambda 捕获变量、move-only 类型等）无法满足 `view` 概念，或者被迫实现一个无用的默认构造函数（往往处于 singular/无效状态）。

### 2. DR20 / P2325R3 的改动

**核心修改**：

- 从 `view` 概念中**移除** `default_initializable` 要求。
- 新定义（C++23 及 C++20 DR）：

```cpp
template <class T>
concept view = range<T> && movable<T> && enable_view<T>;
```

- 同时也从 `weakly_incrementable` 概念中移除了 `default_initializable` 要求（影响某些迭代器）。

**对各个 view 的实际影响**（库实现层面）：

- 许多适配器（如 `filter_view`、`transform_view`、`drop_view`、`join_view` 等）的**默认构造函数**现在被**约束**为：

  ```cpp
  filter_view() requires default_initializable<V> && default_initializable<Pred> = default;
  ```

  只有当内部基 view（`V`）和谓词/函数对象**都能**默认构造时，才提供默认构造函数。

- 这属于“按需”提供默认构造，而不是强制所有 view 都必须默认可构造。

### 3. 为什么这么改？（动机）

- **View 的本质**是“轻量、可廉价移动/复制”的 range。默认构造**不是其核心语义**。
- 很多 view 天然不适合默认构造（例如持有不可默认构造的 lambda、unique_ptr、引用等）。
- 移除这个要求后，能让更多有用的类型成为合法的 `view`，提升 Ranges 库的灵活性。
- 保留 `movable` + `enable_view` 已经足以保证 view 在管道（pipeline）中使用时的性能特性（常数时间 move 等）。

### 4. 实际影响和注意事项

- **向后兼容**：大部分代码不受影响，因为之前能默认构造的 view 仍然可以。
- **新行为**：某些自定义 view 或组合 view 现在能满足 `view` 概念了（之前因为无法默认构造而失败）。
- **标准库 view**：`single_view`、`iota_view` 等仍会在可能时提供受约束的默认构造函数。
- **自定义 view**：如果你实现自己的 view，推荐继承 `view_interface` 并正确 `enable_view`，现在不需要再费力提供默认构造函数。

**cppreference 上的 Defect Report 记录**：

> **P2325R3** applied to C++20: `view` required `default_initializable` → does not require.

### 总结

DR20（P2325）是 Ranges 库的一个重要修复，它**让 `view` 的定义更纯粹、更符合其设计意图**——强调“轻量移动”而非“必须能默认构造”。这是 C++23 对 C++20 的一个重要 DR 回溯修改。

如果你有具体的 view 类型或代码示例想看改动前后的差异，可以再提供，我可以进一步解释。