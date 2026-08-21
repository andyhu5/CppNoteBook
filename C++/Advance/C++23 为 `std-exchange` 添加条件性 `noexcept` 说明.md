# C++23：为 `std::exchange` 添加条件性 `noexcept` 说明（P2401R0）

## 背景：`std::exchange` 是什么

`std::exchange`（C++14 引入）是一个非常常用的工具函数，语义是"**把对象设置为新值，同时返回它的旧值**"：

```cpp
template<class T, class U = T>
T exchange(T& obj, U&& new_value);
```

典型用法（尤其在实现移动构造函数/移动赋值运算符时非常常见）：

```cpp
struct Buffer {
    int* data;
    
    Buffer(Buffer&& other) noexcept
        : data(std::exchange(other.data, nullptr)) // 拿走 other.data，同时把 other.data 置空
    {}
};
```

## 问题：C++20 及之前，`std::exchange` 没有任何 `noexcept` 声明

在 C++23 之前，`std::exchange` 的声明里**完全没有 `noexcept`**，无论 `T` 和 `U` 实际上是否支持不抛异常的移动/赋值操作：

```cpp
template<class T, class U = T>
T exchange(T& obj, U&& new_value); // C++20 及之前：始终不是 noexcept
```

这带来一个实际问题：**`std::exchange` 内部实际执行的操作**（大致等价于）：

```cpp
T old_value = std::move(obj);
obj = std::forward<U>(new_value);
return old_value;
```

如果 `T` 是一个**移动构造/移动赋值都标记为 `noexcept` 的类型**（比如 `int`、`std::unique_ptr<X>`、大多数设计良好的移动语义类型），那么这整个操作**理应也是 `noexcept` 的**——但由于 `std::exchange` 本身的函数签名没有条件性地声明 `noexcept`，编译器和使用者都**无法从类型系统层面得知**这一点。

### 实际影响：破坏了移动构造函数的 `noexcept` 保证链条

```cpp
struct Buffer {
    std::unique_ptr<int[]> data; // 移动操作是 noexcept 的

    Buffer(Buffer&& other) noexcept  // 程序员手动标注了 noexcept
        : data(std::exchange(other.data, nullptr)) // 但 exchange 本身没有 noexcept 保证……
    {}
};
```

在这个例子中，程序员**手动**给移动构造函数标注了 `noexcept`，这是合法的（因为 `unique_ptr` 的相关操作确实不会抛异常），但这依赖于程序员**自己**判断"这里调用的 `exchange` 实际上不会抛"——`std::exchange` 的函数签名本身**没有**通过 `noexcept` 表达这个事实，导致：

- **静态分析工具**、**依赖 `noexcept` 做优化决策的模板代码**（比如 `std::vector` 判断是否要在扩容时用移动而非拷贝，就是通过检查移动操作是否 `noexcept`）无法从 `std::exchange` 的类型信息里推导出它是否安全；
- 一些**通用工具函数**如果内部用了 `std::exchange`，想把自己声明为 `noexcept`（当底层类型支持时），必须**手动列出复杂的 `noexcept` 表达式**去重新推导 `T`/`U` 的移动/赋值操作是否 `noexcept`，非常繁琐、容易出错，且属于对 `std::exchange` 实现细节的重复劳动。

## C++23 的修正：加上条件性 `noexcept`

C++23（P2401R0）给 `std::exchange` 补上了**条件性的 `noexcept` 说明**，大致等价于：

```cpp
template<class T, class U = T>
constexpr T exchange(T& obj, U&& new_value)
    noexcept(std::is_nothrow_move_constructible_v<T> &&
             std::is_nothrow_assignable_v<T&, U>);
```

也就是说：**只有当 `T` 的移动构造是 `noexcept` 的、且 `T` 能以 `noexcept` 的方式从 `U&&` 赋值时，`std::exchange` 本身才被声明为 `noexcept`**；否则它仍然是"可能抛异常"的（`noexcept(false)`）。

## 修正后的效果

```cpp
struct Buffer {
    std::unique_ptr<int[]> data;

    Buffer(Buffer&& other) noexcept
        : data(std::exchange(other.data, nullptr))
        // C++23 起：编译器能自动验证这里的 std::exchange 调用确实是 noexcept 的，
        // 因为 unique_ptr 的移动构造/赋值都是 noexcept
    {}
};

static_assert(noexcept(std::exchange(std::declval<int&>(), 42))); 
// C++23 起：true，因为 int 的移动/赋值显然不抛异常

struct ThrowingType {
    ThrowingType(ThrowingType&&) { /* 可能抛异常，没有 noexcept */ }
    ThrowingType& operator=(ThrowingType&&) { /* 同上 */ }
};

static_assert(!noexcept(std::exchange(std::declval<ThrowingType&>(), ThrowingType{})));
// noexcept(false)，如实反映了底层类型的异常安全性
```

## 为什么这个修正重要

1. **精确性**：`std::exchange` 现在如实反映了它"是否可能抛异常"这一事实，而不是笼统地"永远不承诺不抛"，这让类型系统层面的信息更加准确。
2. **传播性**：任何**依赖 `noexcept` 表达式做条件判断**的代码（比如标准库容器内部判断"移动 vs 拷贝"策略、或者用户自己写的模板代码用 `noexcept(...)` 表达式来推导某个操作是否安全），现在可以正确地把 `std::exchange` 的调用纳入这条推导链，而不需要绕开它、重新手写等价的类型萃取逻辑。
3. **减少样板代码**：库作者在写移动构造/赋值等函数时，如果内部使用了 `std::exchange` 且不想手动指定 `noexcept` 条件表达式，也可以让编译器**自动推导**函数是否应该是 `noexcept`（比如用 `noexcept(noexcept(std::exchange(...)))` 这种"自动传播"写法），不再需要重复分析底层类型属性。
4. **和标准库其他类似工具的一致性**：这符合 C++ 标准库长期以来的一个原则——**只要能够静态判断某个操作绝不抛异常，就应该通过 `noexcept` 表达出来**，让编译器和依赖 `noexcept` 的代码（尤其是容器实现）能做出正确、高效的决策。这类"补上被遗漏的 `noexcept`"的修正在标准库历史上并不罕见（属于持续打磨异常安全性表达的一部分）。

## 小结

这是一项看似很小、但对**泛型代码和异常安全性推导链条**很有价值的修正：`std::exchange` 作为一个被广泛用于移动构造/赋值实现中的工具函数，之前却**没有如实反映其底层操作是否可能抛异常**，导致依赖它的代码要么被迫手动重新推导 `noexcept` 条件（繁琐、易错），要么干脆无法从类型系统获得这个信息。C++23 给它补上条件性 `noexcept` 声明后，`std::exchange` 的异常安全性保证能够**自动、正确地沿着调用链传播**，是标准库持续精细化"什么操作真正保证不抛异常"这一体系的又一处补丁。