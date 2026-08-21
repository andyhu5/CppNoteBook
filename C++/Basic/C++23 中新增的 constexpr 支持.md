# C++23 中新增的 constexpr 支持

C++23 扩大了 `constexpr` 的适用范围，让以下几个标准库组件可以在编译期（常量求值上下文）中使用。逐项解释：

## 1. `std::bitset` 变为 constexpr

C++23 之前，`std::bitset` 的构造函数和大部分成员函数都不是 `constexpr`，无法在编译期使用。C++23 给几乎所有成员函数（构造、`set`/`reset`/`flip`、`test`、`operator[]`、`to_ulong`/`to_ullong`、比较运算符等）加上了 `constexpr`。

```cpp
constexpr std::bitset<8> b{0b1010'1010};
static_assert(b.count() == 4);
```

## 2. `std::unique_ptr` 变为 constexpr

`std::unique_ptr` 的构造、析构、`reset`、`release`、`operator*`、`operator->`、`get` 等操作现在都是 `constexpr`。这意味着可以在编译期动态分配和释放内存（只要该内存在同一次常量求值内被完全释放，不能"泄漏"到运行期）。

```cpp
constexpr int f() {
    auto p = std::make_unique<int>(42); // constexpr new
    return *p;
} // p 在编译期析构，delete 也是 constexpr 的
static_assert(f() == 42);
```

这依赖于 C++20 引入的 **constexpr new/delete**（要求分配的内存必须在同一常量表达式求值期间释放）。

## 3. `std::type_info::operator==` 变为 constexpr

`typeid` 本身在 C++20 起就可以出现在常量表达式里（限于字面类型/多态类型的静态类型比较等场景），C++23 补上了让 `type_info` 的相等比较运算符也是 `constexpr`，使得基于 `typeid` 的编译期类型判断更完整：

```cpp
constexpr bool same = (typeid(int) == typeid(int)); // 现在可行
```

## 4. `<cmath>` 部分函数变为 constexpr

C++23 让 `<cmath>`（以及 `<cstdlib>` 中部分数学相关函数）中很多函数支持 `constexpr`，例如 `std::abs`、`std::fabs`、`std::floor`、`std::ceil`、`std::sqrt`、`std::pow`、三角函数（`sin`/`cos`/`tan` 等）、`std::isnan`/`std::isinf` 等分类函数、`std::lerp` 等。

```cpp
constexpr double r = std::sqrt(4.0); // C++23 起合法
```

注意：并非全部数学函数都被覆盖（依赖具体实现的浮点误差、特殊值处理等因素），标准只要求"在数学上有精确定义、且实现可行"的部分是 `constexpr`，具体范围要查具体函数的 cppreference 页面确认。

## 5. 整数重载的 `std::to_chars` / `std::from_chars` 变为 constexpr

`<charconv>` 中针对**整数类型**（`int`, `long`, `unsigned` 等）的 `to_chars`/`from_chars` 重载在 C++23 中被标记为 `constexpr`，可以在编译期做整数与字符序列之间的转换。

- 注意：**浮点数版本**的 `to_chars`/`from_chars`（针对 `float`/`double`）**仍然不是** `constexpr`，因为其实现依赖了不满足 constexpr 要求的算法/底层库调用。

```cpp
constexpr bool test() {
    char buf[10]{};
    auto res = std::to_chars(buf, buf + 10, 123);
    return std::string_view(buf, res.ptr) == "123";
}
static_assert(test()); // C++23 起合法（仅整数重载）
```

## 总体动机

这些改动都属于 C++23 "让更多标准库设施可用于编译期计算"的整体趋势的一部分，配合 C++20 的 `constexpr` 动态内存分配、`std::is_constant_evaluated`（以及 C++23 的 `if consteval`）等特性，使得诸如**编译期解析、编译期数据结构、编译期字符串处理**等元编程场景变得更容易实现，而不必再依赖手写的、脆弱的 `constexpr` 替代实现。