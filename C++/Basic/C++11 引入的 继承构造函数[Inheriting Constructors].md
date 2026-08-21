这是 C++11 引入的 **继承构造函数（Inheriting Constructors）** 语法。

```cpp
using Base::Base;
```

表示：

> **把基类的所有构造函数引入到当前派生类中。**

你的例子中：

```cpp
struct MyVariant : std::variant<int, std::string>
{
    using variant::variant;

    void hello() const
    {
        std::println("hello");
    }
};
```

由于 `MyVariant` 继承自

```cpp
std::variant<int, std::string>
```

所以

```cpp
using variant::variant;
```

实际上等价于

```cpp
using std::variant<int, std::string>::variant;
```

因为在类体内部，可以直接写基类名 `variant`。

------

## 没有这句会怎样？

例如

```cpp
struct MyVariant : std::variant<int, std::string>
{
};
```

那么

```cpp
MyVariant v = 123;
```

通常不能编译。

原因是：

虽然基类有

```cpp
std::variant<int, std::string>(int);
```

但是**构造函数不会自动继承**。

派生类默认只有：

```cpp
MyVariant();
MyVariant(const MyVariant&);
MyVariant(MyVariant&&);
```

因此

```cpp
MyVariant v = 123;
```

会报错：

```text
no matching constructor
```

------

## 加上 `using variant::variant;`

```cpp
struct MyVariant : std::variant<int, std::string>
{
    using variant::variant;
};
```

编译器会把基类所有可访问的构造函数都"引入"派生类。

相当于自动生成了（概念上）：

```cpp
MyVariant(int x)
    : variant(x)
{
}

MyVariant(const std::string& s)
    : variant(s)
{
}

MyVariant(std::string&& s)
    : variant(std::move(s))
{
}
```

因此：

```cpp
MyVariant a = 42;
MyVariant b = "hello";
MyVariant c = std::string("world");
```

全部合法。

------

## 一个更简单的例子

例如

```cpp
class Animal
{
public:
    Animal(std::string name)
    {
        std::println("{}", name);
    }
};
```

派生：

```cpp
class Dog : public Animal
{
public:
    using Animal::Animal;
};
```

现在：

```cpp
Dog d("Lucky");
```

等价于编译器帮你生成了：

```cpp
class Dog : public Animal
{
public:
    Dog(std::string name)
        : Animal(name)
    {
    }
};
```

输出

```text
Lucky
```

------

## 它不会继承普通成员函数

很多初学者容易误会：

```cpp
using Animal::Animal;
```

**仅继承构造函数。**

它不会影响：

```cpp
class Animal
{
public:
    void eat();
};
```

因为普通成员函数本来就已经继承了。

`using Base::Base` 的特殊之处就在于：

> **构造函数默认是不继承的。**

------

## 为什么 `std::variant` 经常这样写？

例如：

```cpp
struct Json
    : std::variant<
        std::nullptr_t,
        bool,
        int,
        double,
        std::string>
{
    using variant::variant;

    bool isNumber() const
    {
        return std::holds_alternative<int>(*this) ||
               std::holds_alternative<double>(*this);
    }
};
```

这样既保留了 `std::variant` 的所有构造能力：

```cpp
Json j1 = 10;
Json j2 = 3.14;
Json j3 = "hello";
```

又可以增加自己的接口：

```cpp
j1.isNumber();
```

这是继承 `std::variant` 最常见的写法，也是 **C++23 DR17** 支持 `std::visit` 派生类的重要应用场景。

------

### 总结

`using Base::Base;`（这里即 `using variant::variant;`）的语义是：

- 将**基类的所有构造函数**引入派生类。
- 派生类可以像基类一样使用这些构造函数，而无需逐个编写转发构造函数。
- 它**只影响构造函数**，不会影响普通成员函数、析构函数或赋值运算符。
- 在继承 `std::variant`、`std::exception`、`std::runtime_error` 等标准库类型时，这是一种非常常见且推荐的写法。