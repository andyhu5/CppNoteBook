**Make `()` more optional for lambda expressions** 是 C++26 的一项语法改进（提案 **P1102R2**），目标是**减少 Lambda 的样板代码（boilerplate）**，让最简单的 Lambda 写起来更自然。

## C++23 以前

Lambda 的基本语法是：

```cpp
[captures] () specifiers -> return_type {
    body
}
```

即使没有参数，也经常需要写一对 `()`：

```cpp
auto f = []() {
    std::puts("Hello");
};
```

如果还要写 `mutable`、`constexpr`、`noexcept`、模板参数等，语法会越来越长。

不过，C++23 已经允许在**没有任何 lambda-specifier 时**省略 `()`：

```cpp
auto f = [] {
    std::puts("Hello");
};
```

但是，一旦出现任何说明符，就必须重新写 `()`。

例如：

```cpp
auto f = [] constexpr { };    // C++23 错误
```

必须写成

```cpp
auto f = []() constexpr { };
```

这是很多人觉得不一致的地方。

------

## C++26 改进

C++26 进一步放宽规则：

**只要 Lambda 没有参数，就可以省略 `()`，即使后面还有说明符。**

例如：

### constexpr

C++23

```cpp
auto f = []() constexpr {
    return 42;
};
```

C++26

```cpp
auto f = [] constexpr {
    return 42;
};
```

------

### mutable

C++23

```cpp
auto f = [x]() mutable {
    ++x;
};
```

C++26

```cpp
auto f = [x] mutable {
    ++x;
};
```

------

### noexcept

C++23

```cpp
auto f = []() noexcept {
};
```

C++26

```cpp
auto f = [] noexcept {
};
```

------

### consteval

C++23

```cpp
auto f = []() consteval {
    return 1;
};
```

C++26

```cpp
auto f = [] consteval {
    return 1;
};
```

------

### static

C++23

```cpp
auto f = []() static {
};
```

C++26

```cpp
auto f = [] static {
};
```

------

## 有参数时仍不能省略

例如

```cpp
auto f = [](int x) {
    return x * 2;
};
```

参数列表当然不能省略，否则编译器不知道参数是什么。

同样，

```cpp
[]<typename T>(T x) { }
```

模板参数和函数参数都存在，因此也不能省略参数列表。

------

## 为什么这样修改？

主要原因有三个：

**1. 消除语法不一致**

以前：

```cpp
[] { }              // 可以
[]() { }            // 可以

[] constexpr { }    // 不可以
[]() constexpr { }  // 可以
```

看起来没有技术上的必要，只是语法限制。

现在统一成：

```cpp
[] { }
[] constexpr { }
[] mutable { }
[] noexcept { }
[] static { }
```

都采用相同风格。

**2. 减少样板代码**

很多 Lambda 都没有参数：

```cpp
std::ranges::for_each(v, [] {
    std::puts("Hi");
});
```

相比

```cpp
std::ranges::for_each(v, []() {
    std::puts("Hi");
});
```

少了一对无意义的 `()`。

**3. 与现代 C++ 的简洁风格一致**

近年来 C++ 持续减少冗余语法，例如：

- `if (init; cond)`
- 缩短模板参数写法
- `deducing this`
- 更简洁的 Lambda

这一改动也是同一方向上的演进。

## 哪些情况可以省略 `()`？

| Lambda 形式       | C++23             | C++26             |
| ----------------- | ----------------- | ----------------- |
| `[] {}`           | ✔                 | ✔                 |
| `[] constexpr {}` | ✘                 | ✔                 |
| `[] mutable {}`   | ✘                 | ✔                 |
| `[] noexcept {}`  | ✘                 | ✔                 |
| `[] consteval {}` | ✘                 | ✔                 |
| `[] static {}`    | ✘                 | ✔                 |
| `[](int x){}`     | ✔（不能省略参数） | ✔（不能省略参数） |

总结来说，**C++26 并没有改变 Lambda 的语义或性能，只是放宽了语法限制：对于没有参数的 Lambda，即使后面跟着 `constexpr`、`mutable`、`noexcept`、`consteval`、`static` 等说明符，也可以省略空参数列表 `()`，使代码更简洁且语法更加一致。**

