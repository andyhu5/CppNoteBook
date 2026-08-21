`const&`-qualified pointer to members 是 **C++20** 对 **pointer-to-member（成员指针）** 的一个语言修正，来源于 WG21 提案 **P0704R1**。它解决了成员函数引用限定符（ref-qualifier）与成员指针类型表示不一致的问题。

## 背景

C++11 引入了 **ref-qualifier**，允许成员函数根据对象是左值还是右值进行重载。

例如：

```cpp
struct S
{
    void f() &        { }   // 只能用于左值对象
    void f() &&       { }   // 只能用于右值对象
};
```

调用：

```cpp
S s;

s.f();      // 调用 f() &
S{}.f();    // 调用 f() &&
```

------

## 成员函数类型实际上包含 ref-qualifier

例如：

```cpp
void S::f() &
```

真正的函数类型可以理解为

```cpp
void () &
```

而

```cpp
void S::f() &&
```

类型则是

```cpp
void () &&
```

因此成员函数指针实际上也应该保存这些限定符。

例如

```cpp
void (S::*pf)() & = &S::f;
```

这里的

```cpp
void (S::*)() &
```

就是完整类型。

------

## C++17以前的问题

对于普通函数类型，可以写

```cpp
using F = void() &;
```

但是不能写

```cpp
using F = const void() &;    // 非法
```

同样，对于成员函数指针，也无法很好地表达某些 cv/ref 限定组合。

例如下面这种类型系统并不完整：

```cpp
void (S::*)() &
void (S::*)() &&
void (S::*)() const
```

但

```cpp
void (S::*)() const &
```

在很多场景下不能一致地作为完整函数类型处理。

模板元编程中尤其容易遇到问题。

------

# C++20 改进

C++20 统一了函数类型与成员函数指针类型。

现在可以完整表示：

```cpp
void (S::*)() const &
```

例如

```cpp
struct S
{
    void f() const &
    {
    }
};

int main()
{
    auto p = &S::f;

    static_assert(
        std::is_same_v<
            decltype(p),
            void (S::*)() const &
        >);
}
```

现在这是标准保证的。

------

## 模板中的意义

以前写 traits 很麻烦：

```cpp
template<typename T>
struct member_traits;
```

需要分别偏特化：

```cpp
R (C::*)()

R (C::*)() const

R (C::*)() &

R (C::*)() &&

...
```

而对于

```cpp
const &
```

这种组合并不能统一处理。

C++20以后可以写

```cpp
template<typename R, typename C>
struct member_traits<R (C::*)() const &>
{
};
```

能够正确匹配。

例如：

```cpp
struct X
{
    int foo() const &
    {
        return 0;
    }
};

template<typename T>
struct traits;

template<typename R, typename C>
struct traits<R (C::*)() const &>
{
    static constexpr bool value = true;
};

static_assert(traits<decltype(&X::foo)>::value);
```

------

# 与 `std::invoke`

`std::invoke` 本身已经支持：

```cpp
struct S
{
    void f() const &
    {
        std::cout << "const&\n";
    }
};

int main()
{
    auto p = &S::f;

    const S s;

    std::invoke(p, s);
}
```

这里

```cpp
p
```

的类型就是

```cpp
void (S::*)() const &
```

C++20使这种类型系统更加一致。

------

# 与普通函数区别

注意：

普通函数不能有

```cpp
void f() const;
```

因为 `const`、`&`、`&&` 是**成员函数限定符**，用于限定隐式对象参数（`*this`），普通函数没有 `this` 指针，因此不能使用这些限定符。

所以

```cpp
void() const &
```

不是合法的普通函数类型。

而

```cpp
void (S::*)() const &
```

是合法的成员函数指针类型。

------

# 实际价值

这个特性主要影响**泛型库作者**，普通应用代码几乎感觉不到变化。

它带来的好处包括：

- 成员函数指针类型与成员函数声明保持一致。
- 支持完整的 `const`、`volatile`、`&`、`&&` 组合。
- 简化模板偏特化和类型萃取（Traits）的实现。
- 改善 `decltype(&Class::member)` 的类型一致性。
- 为 `std::invoke`、`std::mem_fn`、`std::bind` 等泛型调用工具提供更完善的类型支持。

对于编写反射库、序列化框架、信号槽系统或其他依赖成员函数指针进行类型推导的高级 C++ 库而言，这一改进消除了此前类型系统中的一些不一致，使模板代码更自然、更易维护。