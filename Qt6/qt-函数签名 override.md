`override` 是 C++11 引入的一个**函数覆盖说明符（override specifier）**，它告诉编译器："**这个函数必须覆盖（override）基类中的一个虚函数**"。

对于你的代码：

```cpp
class TableModel : public QAbstractTableModel
{
public:
    int rowCount(const QModelIndex &parent) const override;
    int columnCount(const QModelIndex &parent) const override;
};
```

这里的 `override` 表示 `rowCount()` 和 `columnCount()` 必须覆盖 `QAbstractItemModel` 中对应的虚函数。

## 不加 `override`

代码依然可以正常编译（前提是函数签名正确）。

```cpp
int rowCount(const QModelIndex &parent) const;
```

如果签名完全一致，它仍然会覆盖基类的虚函数。

例如：

```cpp
class Base
{
public:
    virtual void foo(int);
};

class Derived : public Base
{
public:
    void foo(int);      // 仍然是 override
};
```

因此：

> **是否真正覆盖虚函数，与 `override` 无关。**

------

## 加 `override`

编译器会额外检查：

> **这个函数是否真的覆盖了某个基类虚函数。**

如果没有覆盖成功，直接编译报错。

例如：

```cpp
class Base
{
public:
    virtual void foo(int);
};

class Derived : public Base
{
public:
    void foo(double) override;
};
```

编译器会报类似错误：

```
error: 'foo' marked override but does not override
```

因为：

```
Base:
foo(int)

Derived:
foo(double)
```

参数不同，不是同一个函数。

------

## 为什么推荐一定写 `override`

Qt 开发中非常容易因为签名写错而没有真正覆盖。

例如：

正确的是：

```cpp
int rowCount(const QModelIndex &parent) const override;
```

如果写成：

```cpp
int rowCount(QModelIndex &parent) const;
```

少了 `const`：

```
const QModelIndex &
↓

QModelIndex &
```

实际上这是两个不同的函数。

没有 `override`：

```
✓ 编译成功
```

但是：

```
QAbstractItemModel
        │
        ├── rowCount(const QModelIndex&)
        │
        └── TableModel
             rowCount(QModelIndex&)
```

Qt 调用的是父类那个函数，你自己的根本不会执行。

结果就是：

```
TableView
↓

rowCount()

↓

返回 0

↓

界面空白
```

这种错误非常难排查。

------

如果加了 `override`：

```cpp
int rowCount(QModelIndex &parent) const override;
```

立即报错：

```
error:
'rowCount' marked override,
but does not override
```

一下就定位问题。

------

## 再举几个常见错误

### ① 少了 const

```cpp
// 基类
virtual int rowCount(const QModelIndex &) const;

// 子类
int rowCount(const QModelIndex &);
```

没有 `const`。

没有 `override`

```
✓ 编译
```

有 `override`

```
✗ 编译失败
```

------

### ② 参数类型错误

```cpp
virtual QVariant data(const QModelIndex &, int role) const;
```

写成

```cpp
QVariant data(QModelIndex index, int role) const override;
```

虽然这里按值传递与按引用在重载中属于不同签名，因此不会覆盖。`override` 会立即报错。

------

### ③ 返回类型错误

```cpp
virtual int rowCount(...) const;
```

写成

```cpp
long rowCount(...) const override;
```

也会报错。

------

## `override` 不会影响运行效率

很多人担心：

```
override 会不会增加性能开销？
```

答案是：

**完全不会。**

它只是编译期检查。

生成的机器码与不写 `override` 完全一样。

------

## Qt 官方建议

Qt 官方以及现代 C++（C++11 以后）的编码规范都建议：

> **所有覆盖基类虚函数的成员函数都应显式写上 `override`。**

例如 Qt 源码中大量都是：

```cpp
bool event(QEvent *event) override;
void paintEvent(QPaintEvent *event) override;
QVariant data(const QModelIndex &, int) const override;
int rowCount(const QModelIndex &) const override;
```

## 总结

| 是否加 `override` | 能否覆盖虚函数     | 编译器检查                           | 推荐程度 |
| ----------------- | ------------------ | ------------------------------------ | -------- |
| 不加              | 可以（签名正确时） | 不检查，容易因签名错误而未覆盖       | 不推荐   |
| 加 `override`     | 可以（签名正确时） | 检查是否真正覆盖，错误会在编译期发现 | 强烈推荐 |

现代 C++ 项目（包括 Qt 6）中，几乎可以将 `override` 视为覆盖虚函数时的标准写法。