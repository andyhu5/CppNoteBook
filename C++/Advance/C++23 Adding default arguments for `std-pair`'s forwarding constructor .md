**Adding default arguments for `std::pair`'s forwarding constructor** 是 **C++23** 的一项标准库改进，来源于提案 **P1951R1**。



正确的用法：

```c
#include <utility>
#include <string>

int main(int argc, char const *argv[])
{
    std::pair<int, std::string> p({},"hello");
    return 0;
}
```



它解决了 `std::pair` 一个长期存在但容易忽略的易用性问题：**转发构造函数（forwarding constructor）现在支持默认参数**，因此可以只提供第一个元素、第二个元素，或者一个都不提供。

------

## C++20 之前的问题

`std::pair` 有一个转发构造函数，大致如下（简化）：

```cpp
template<class U1, class U2>
pair(U1&& x, U2&& y);
```

两个参数都是必须的。

例如：

```cpp
std::pair<int, std::string> p(1, "hello");
```

没有问题。

但是：

```cpp
std::pair<int, std::string> p(1);
```

编译失败。

因为：

```text
缺少第二个参数
```

同样：

```cpp
std::pair<int, std::string> p;
```

虽然可以调用默认构造函数，但**一旦编译器选择的是转发构造函数，就不能只传一个参数**。

------

## 为什么这是个问题？

假设：

```cpp
std::pair<int, std::string> p(42);
```

很多人直觉认为应该得到：

```text
first  = 42
second = ""
```

但实际上 C++20 不允许。

你只能写：

```cpp
std::pair<int, std::string> p(42, "");
```

或者：

```cpp
std::pair<int, std::string> p{42, {}};
```

需要手动补上第二个默认值。

------

## C++23 的改进

C++23 将转发构造函数修改为（简化）：

```cpp
template<class U1 = T1, class U2 = T2>
pair(U1&& x = U1{}, U2&& y = U2{});
```

实际标准实现比这复杂得多（涉及约束和 SFINAE），但语义可以理解为：

> **两个参数都拥有默认值。**

因此下面都合法。

------

### 一个参数

```cpp
#include <utility>
#include <string>

std::pair<int, std::string> p(42);
```

结果：

```text
first  = 42
second = ""
```

第二个成员使用默认构造。

------

### 零参数

```cpp
std::pair<int, std::string> p;
```

结果：

```text
first  = 0
second = ""
```

虽然以前也能默认构造，但现在统一由同一套构造机制处理。

------

### 两个参数

当然仍然可以：

```cpp
std::pair<int, std::string> p(42, "hello");
```

得到：

```text
first  = 42
second = "hello"
```

------

## 再举几个例子

### `std::pair<int, double>`

```cpp
std::pair<int, double> p(5);
```

结果：

```text
first  = 5
second = 0.0
```

------

### `std::pair<std::string, std::vector<int>>`

```cpp
std::pair<std::string, std::vector<int>> p("abc");
```

结果：

```text
first  = "abc"
second = {}
```

第二个成员为空 `vector`。

------

### `std::pair<std::unique_ptr<int>, int>`

```cpp
std::pair<std::unique_ptr<int>, int> p(
    std::make_unique<int>(10));
```

结果：

```text
first  = unique_ptr(...)
second = 0
```

------

## 为什么要修改？

### 1. 更符合直觉

很多标准库类型都支持：

```cpp
Type(value)
```

表示：

```text
第一个成员初始化
其他成员默认构造
```

`std::pair` 却不能。

------

### 2. 与 `std::tuple` 保持一致

例如：

```cpp
std::tuple<int, std::string> t(1);
```

早就支持：

```text
1
""
```

而：

```cpp
std::pair<int, std::string> p(1);
```

却不支持。

C++23 统一了二者行为。

------

### 3. 泛型代码更容易编写

例如：

```cpp
template<class T>
T make()
{
    return T(42);
}
```

对于：

```cpp
using T = std::pair<int, std::string>;
```

C++20：

```text
编译失败
```

C++23：

```text
成功
```

------

## 实际影响

这项修改**不会影响已有代码**。

例如：

```cpp
std::pair<int, int> p(1, 2);
```

行为完全不变。

它只是让以前不能编译的代码：

```cpp
std::pair<int, std::string> p(1);
```

现在可以编译。

------

## 与聚合初始化的区别

以前有人会写：

```cpp
std::pair<int, std::string> p{1};
```

有些情况下可以工作，因为使用的是聚合初始化或其他构造函数。

而：

```cpp
std::pair<int, std::string> p(1);
```

却失败。

C++23 后，两种形式的行为更加一致，都可以只提供一个参数。

------

## 总结

| 写法                                                     | C++20                 | C++23                    |
| -------------------------------------------------------- | --------------------- | ------------------------ |
| `std::pair<int,std::string> p();`                        | ✔                     | ✔                        |
| `std::pair<int,std::string> p(1, "abc");`                | ✔                     | ✔                        |
| `std::pair<int,std::string> p(1);`                       | ❌                     | ✔                        |
| `std::pair<int,std::string> p("abc");`（对应首元素类型） | ✔/❌（取决于类型匹配） | 更一致地支持默认第二参数 |
| 未提供的成员                                             | —                     | 默认构造                 |

**一句话概括：** C++23 为 `std::pair` 的转发构造函数增加了默认参数，使其可以像 `std::tuple` 一样，只初始化前面的成员，而未提供的成员自动进行默认构造。这提高了 `std::pair` 的易用性，也让它与其他标准库类型的构造行为更加一致。