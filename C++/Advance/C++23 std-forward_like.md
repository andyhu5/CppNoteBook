`std::forward_like` 是 **C++23** 在 `<utility>` 中新增的工具函数，来源于提案 **P2445R1**。

它解决了一个长期存在的问题：

> **"像另一个类型一样进行完美转发（forward）"。**

简单来说：

- `std::move`：总是转成右值。
- `std::forward<T>`：根据模板参数 `T` 转发。
- **`std::forward_like<T>`：让一个对象拥有和 `T` 相同的 `const` 和引用（value category）属性。**

------

## 为什么需要它？

先回顾 `std::forward`。

```cpp
template<typename T>
void foo(T&& x)
{
    bar(std::forward<T>(x));
}
```

这里：

- `T = int&` → 转发为 `int&`
- `T = int` → 转发为 `int&&`

这是标准的 Perfect Forwarding。

但是很多时候，我们没有一个模板参数 `T`，而是希望：

> **"把对象 y 转发成和对象 x 一样的引用类型。**

例如：

```cpp
struct Wrapper
{
    T value;
};
```

如果

```cpp
Wrapper&
```

调用：

```cpp
wrapper.value
```

应该得到：

```cpp
T&
```

如果

```cpp
Wrapper&&
```

调用：

```cpp
wrapper.value
```

应该得到：

```cpp
T&&
```

以前实现起来很麻烦。

------

## 没有 `forward_like`

例如：

```cpp
template<typename Self>
decltype(auto) get(Self&& self)
{
    return std::forward<Self>(self).value;
}
```

很多人会写：

```cpp
return std::forward<Self>(self.value);
```

这是错误的。

因为

```cpp
self.value
```

不是一个独立的模板参数。

------

以前通常要写：

```cpp
return static_cast<
    copy_cvref_t<Self, decltype(self.value)>
>(self.value);
```

其中

```cpp
copy_cvref_t
```

是一堆复杂的模板元编程。

Ranges 库里面大量都是这种代码。

------

## C++23：`std::forward_like`

现在：

```cpp
#include <utility>

return std::forward_like<Self>(self.value);
```

即可。

------

## 一个简单例子

```cpp
#include <utility>

int x = 10;

int& l = std::forward_like<int&>(x);

int&& r = std::forward_like<int&&>(x);
```

结果：

第一句得到：

```text
int&
```

第二句得到：

```text
int&&
```

即：

**按照模板参数的引用属性进行转发。**

------

## 它复制哪些属性？

`forward_like<T>(x)` 会复制 `T` 的：

- `const`
- `volatile`
- 左值引用
- 右值引用

例如：

| Like 类型     | 返回类型    |
| ------------- | ----------- |
| `int&`        | `U&`        |
| `const int&`  | `const U&`  |
| `int&&`       | `U&&`       |
| `const int&&` | `const U&&` |

这里：

```text
U
```

就是

```cpp
decltype(x)
```

去掉引用后的类型。

------

## 举几个例子

### 左值

```cpp
int x;

auto&& a =
    std::forward_like<int&>(x);
```

得到：

```cpp
int&
```

------

### const 左值

```cpp
int x;

auto&& a =
    std::forward_like<const int&>(x);
```

得到：

```cpp
const int&
```

------

### 右值

```cpp
int x;

auto&& a =
    std::forward_like<int&&>(x);
```

得到：

```cpp
int&&
```

------

### const 右值

```cpp
int x;

auto&& a =
    std::forward_like<const int&&>(x);
```

得到：

```cpp
const int&&
```

------

## 一个典型用途

假设：

```cpp
template<typename T>
struct Box
{
    T value;

    template<typename Self>
    decltype(auto) get(this Self&& self)
    {
        return std::forward_like<Self>(self.value);
    }
};
```

现在：

```cpp
Box<int> b;

b.get();
```

返回：

```cpp
int&
```

而：

```cpp
std::move(b).get();
```

返回：

```cpp
int&&
```

完全符合对象本身的值类别。

如果没有：

```cpp
std::forward_like
```

这段代码要写几十行模板。

------

## 与 Deducing this 配合

这是 `forward_like` 最重要的用途。

例如 C++23：

```cpp
template<typename Self>
decltype(auto) data(this Self&& self)
{
    return std::forward_like<Self>(self.buffer_);
}
```

其中：

```cpp
Self&
```

得到：

```cpp
buffer_&
```

而：

```cpp
Self&&
```

得到：

```cpp
buffer_&&
```

这正是 `Deducing this`（显式对象参数）设计所需要的。

------

## 与 `std::forward` 的区别

假设：

```cpp
template<typename T>
void foo(T&& x)
{
    std::forward<T>(x);
}
```

这里：

`forward` 的依据是：

```text
模板参数 T
```

而：

```cpp
std::forward_like<T>(y);
```

依据的是：

```text
模板参数 T
```

但**作用对象是 y**。

即：

```text
T
↓
复制 cv/ref
↓
y
```

因此：

```text
forward
```

关注：

> **转发当前参数。**

而：

```text
forward_like
```

关注：

> **把另一个对象转发得像 T 一样。**

------

## 与 `std::move` 的区别

| 函数                      | 左值     | 右值     | const                    |
| ------------------------- | -------- | -------- | ------------------------ |
| `std::move(x)`            | 总变右值 | 总变右值 | 保留 const               |
| `std::forward<T>(x)`      | 依据 `T` | 依据 `T` | 保留                     |
| `std::forward_like<T>(y)` | 依据 `T` | 依据 `T` | 保留，并作用于另一个对象 |

------

## 实现原理（简化）

标准实现大致相当于：

```cpp
template<class T, class U>
constexpr decltype(auto) forward_like(U&& x)
{
    // 根据 T 复制 const/volatile 和引用属性，
    // 然后返回 x。
}
```

其核心思想就是：

1. 去掉 `T` 和 `U` 的引用。
2. 将 `T` 的 `const`/`volatile` 限定复制到 `U`。
3. 如果 `T` 是左值引用，则返回左值引用；如果 `T` 是右值引用，则返回右值引用。

真正的标准实现还会正确处理 `volatile` 等细节，但概念就是如此。

## 总结

| 特性              | `std::forward_like`                          |
| ----------------- | -------------------------------------------- |
| C++版本           | C++23                                        |
| 头文件            | `<utility>`                                  |
| 是否 `constexpr`  | ✔                                            |
| 是否 `noexcept`   | ✔                                            |
| 保留 `const`      | ✔                                            |
| 保留 `volatile`   | ✔                                            |
| 保留左值/右值属性 | ✔                                            |
| 主要用途          | 将一个对象按另一个类型的 cv/ref 属性进行转发 |

**一句话概括：** `std::forward_like<T>(x)` 的作用是**把对象 `x` 转发成与类型 `T` 相同的 `const`/`volatile` 和引用类别（左值或右值）**。它是 `std::forward` 的补充，特别适合与 C++23 的 **Deducing this**、Ranges 和各种泛型包装器配合使用，大幅简化了过去需要复杂模板元编程才能实现的“复制 cv/ref 属性”问题。