这是 **C++23 DR17** 的一个修复，完整标题通常写作：

> **DR17: `std::visit` for classes derived from `std::variant`**

它修复了 **C++17/C++20 中 `std::visit` 对继承自 `std::variant` 的类支持不完整**的问题。

一句话概括：

> **C++20 中，`std::visit` 只能可靠地接受 `std::variant` 本身；C++23 DR17 扩展为也支持继承自 `std::variant` 的类型。**

------

## 1. `std::visit` 回顾

例如：

```cpp
std::variant<int, std::string> v = 42;

std::visit([](auto&& x) {
    std::println("{}", x);
}, v);
```

输出

```text
42
```

这里 `visit` 根据当前存储的类型，调用对应的 lambda。

------

## 2. C++20 的问题

假设你希望扩展 `variant`：

```cpp
struct MyVariant : std::variant<int, std::string>
{
    using variant::variant;

    void print() const
    {
        std::println("MyVariant");
    }
};
```

然后：

```cpp
MyVariant v = 100;

std::visit([](auto&& x) {
    std::println("{}", x);
}, v);
```

理论上应该输出：

```text
100
```

但是 C++20 标准规定：

```cpp
std::visit
```

参数要求是

```text
variant<Ts...>
```

而不是

```text
derived from variant
```

很多实现因此拒绝编译。

例如：

```text
error:
no matching function for std::visit(...)
```

------

## 为什么会这样？

`std::visit` 内部需要使用很多 `variant` 的辅助函数，例如：

```cpp
v.index();
```

以及：

```cpp
std::get<0>(v);
```

标准最初把参数限定成：

```cpp
variant<Ts...>&
```

没有考虑继承关系。

即使：

```cpp
struct MyVariant
    : std::variant<int, std::string>
{
};
```

它依然不是：

```cpp
std::variant<int,std::string>
```

而只是它的子类。

------

## 3. DR17 如何修复？

DR17 修改了 `visit` 的规范：

以前：

```text
visit()

↓

只接受 variant
```

现在：

```text
visit()

↓

接受 variant
↓

以及公开继承 variant 的类
```

也就是说：

```cpp
struct MyVariant :
    std::variant<int,std::string>
{
    using variant::variant;
};
```

现在：

```cpp
MyVariant v = "Hello";

std::visit([](auto&& x)
{
    std::println("{}", x);
}, v);
```

合法。

输出：

```text
Hello
```

------

## 4. 为什么要支持继承？

有时候，希望给 `variant` 增加成员函数。

例如：

```cpp
struct JsonValue
    : std::variant<
        nullptr_t,
        bool,
        int,
        double,
        std::string>
{
    using variant::variant;

    bool isNumber() const
    {
        return std::holds_alternative<int>(*this)
            || std::holds_alternative<double>(*this);
    }
};
```

然后：

```cpp
JsonValue j = 123;
```

可以：

```cpp
j.isNumber();
```

同时仍然：

```cpp
std::visit(..., j);
```

如果没有 DR17：

```text
继承以后

↓

visit 不能用了
```

就很尴尬。

------

## 5. DR17 之后

例如：

```cpp
#include <variant>
#include <print>

struct MyVariant
    : std::variant<int, std::string>
{
    using variant::variant;

    void hello() const
    {
        std::println("hello");
    }
};

int main()
{
    MyVariant v = "C++23";

    std::visit([](const auto& x)
    {
        std::println("{}", x);
    }, v);

    v.hello();
}
```

输出：

```text
C++23
hello
```

完全合法。

------

## 6. 对实现有什么影响？

DR17 修改了标准库内部的约束。

以前很多实现类似：

```cpp
template<class Visitor, class... Types>
visit(Visitor&&,
      variant<Types...>&);
```

现在概念上更接近：

```cpp
template<class Visitor, class Variant>
visit(Visitor&&,
      Variant&&);
```

然后要求：

```text
Variant

↓

公开继承 variant
```

即可。

标准库内部会把它转换成真正的 `variant` 来处理。

------

## 总结

**C++23 DR17** 并没有引入新的 API，而是修复了 `std::visit` 的一个设计限制。

- **C++20**：`std::visit` 只能可靠地接受 `std::variant` 对象，继承自 `std::variant` 的类可能无法编译。
- **C++23 DR17**：`std::visit` 扩展为支持 **公开继承自 `std::variant`** 的类型。
- **主要受益场景**：为 `std::variant` 添加成员函数、业务逻辑或类型别名，构建更符合领域模型的封装类型，而不会失去 `std::visit`、`std::get`、`std::holds_alternative` 等标准库功能。

这是一个典型的 Defect Report：**没有增加新的语言能力，而是让标准库的行为更符合开发者的直觉和继承语义。**