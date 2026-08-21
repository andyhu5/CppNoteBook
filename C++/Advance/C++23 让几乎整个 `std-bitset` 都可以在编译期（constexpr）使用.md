`constexpr std::bitset` 是 **C++23** 的一项标准库增强（提案 **P2417R2**）。

一句话概括：

> **C++23 让几乎整个 `std::bitset` 都可以在编译期（constexpr）使用。**

这是一项非常实用的改进，因为 `bitset` 本质上是一个固定大小的位数组，非常适合在编译期进行位运算。

------

## 1. C++20 存在什么问题？

先看一个简单例子：

```cpp
constexpr std::bitset<8> bs("10110011");
```

在 C++20 中，这通常不能工作，因为：

- `std::bitset` 的很多构造函数不是 `constexpr`
- `set()`
- `reset()`
- `flip()`
- `count()`
- `test()`

等等，大多数成员函数都不能在常量表达式中调用。

例如：

```cpp
constexpr int f()
{
    std::bitset<8> bs;
    bs.set(3);
    return bs.count();
}
```

C++20：

```
error:
call to non-constexpr function
```

因此：

```text
bitset

理论上适合编译期

↓

实际上只能运行期使用
```

------

# 2. C++23 做了什么？

C++23 将几乎所有成员函数都改成了 `constexpr`。

包括：

```cpp
bitset::set()

bitset::reset()

bitset::flip()

bitset::count()

bitset::all()

bitset::any()

bitset::none()

bitset::test()

bitset::operator[]

bitset::to_ulong()

bitset::to_ullong()

位运算符

构造函数
```

基本可以认为：

> **整个 `std::bitset` 都变成了 constexpr 容器。**

------

# 3. 一个典型例子

现在：

```cpp
constexpr int f()
{
    std::bitset<8> bs;

    bs.set(0);
    bs.set(3);
    bs.flip(1);

    return bs.count();
}

static_assert(f() == 3);
```

C++23：

完全合法。

整个函数：

```text
编译期间执行

↓

count()

↓

3

↓

static_assert 成功
```

------

# 4. 编译期位运算

例如：

```cpp
constexpr auto f()
{
    std::bitset<8> a("10101010");
    std::bitset<8> b("11110000");

    return a & b;
}

constexpr auto x = f();

static_assert(x.to_ulong() == 0b10100000);
```

以前：

```
operator&
```

不是 constexpr。

现在：

全部可以。

------

# 5. 编译期生成 Mask

这是最常见用途。

例如：

```cpp
constexpr auto make_mask()
{
    std::bitset<32> mask;

    for (int i = 0; i < 32; i += 2)
        mask.set(i);

    return mask;
}

constexpr auto mask = make_mask();
```

整个：

```text
for

↓

set

↓

bitset

↓

编译期间完成
```

运行时没有任何开销。

------

# 6. 编译期权限系统

例如：

```cpp
enum Permission
{
    Read,
    Write,
    Execute,
    Delete
};
```

可以：

```cpp
constexpr auto admin()
{
    std::bitset<4> p;

    p.set(Read);
    p.set(Write);
    p.set(Execute);
    p.set(Delete);

    return p;
}

constexpr auto guest()
{
    std::bitset<4> p;

    p.set(Read);

    return p;
}
```

然后：

```cpp
static_assert(admin().count() == 4);
static_assert(guest().count() == 1);
```

全部发生在编译期间。

------

# 7. 编译期查表

例如：

ASCII 是否为数字：

```cpp
constexpr auto make_table()
{
    std::bitset<128> table;

    for(char c='0'; c<='9'; ++c)
        table.set(c);

    return table;
}

constexpr auto digits = make_table();

static_assert(digits.test('5'));
static_assert(!digits.test('A'));
```

整个查找表：

```text
编译期间生成

↓

运行时直接访问
```

以前这种工作通常要自己写 `constexpr std::array<bool,128>`。

现在：

```cpp
std::bitset<128>
```

更紧凑，也更适合位运算。

------

# 8. 为什么值得做？

`bitset` 有几个天然优势：

- 固定大小（模板参数 `N`）
- 不动态分配内存
- 存储连续位
- 所有操作都是纯计算

因此它非常适合 `constexpr`。

在 C++20 中，很多标准库容器因为涉及动态内存，还无法完全 constexpr；而 `bitset` 几乎没有这些限制，因此 C++23 将它全面 constexpr 化。

------

# 9. 与 `std::vector<bool>` 的区别

例如：

```cpp
std::vector<bool>
```

虽然也是位压缩，但：

- 动态分配内存
- 长度运行时决定
- C++23 仍不能普遍用于编译期

而：

```cpp
std::bitset<64>
```

具有：

- 固定大小
- 无堆分配
- C++23 几乎所有操作都是 `constexpr`

因此，如果位数在编译期已知，`std::bitset` 是进行编译期位操作的首选。

------

## 总结

C++23 的 **`constexpr std::bitset`** 并没有引入新的接口，而是将 `std::bitset` 的绝大多数成员函数和运算符都声明为 `constexpr`。

主要收益包括：

- **编译期构造**：可以在 `constexpr` 函数中创建和初始化 `bitset`。
- **编译期修改**：支持 `set()`、`reset()`、`flip()` 等修改操作。
- **编译期查询**：支持 `count()`、`test()`、`all()`、`any()`、`none()` 等查询。
- **编译期位运算**：支持 `&`、`|`、`^`、`~`、移位等运算。
- **典型应用**：生成位掩码（bit mask）、权限位、查找表、状态机、协议标志等，全部可在编译期间完成，从而消除运行时初始化开销。

因此，`std::bitset` 在 C++23 中真正成为了一个**适用于编译期元编程和运行期高效位操作**的标准容器。