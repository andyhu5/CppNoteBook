C++23 attributes on lambda-expressions

## 这个特性解决的问题

lambda 的语法里，原本只保留了一个能写属性的位置——参数列表之后、紧跟函数体之前。这个位置写出的属性，按语言规则归属于函数调用运算符的*类型*，而不是函数调用运算符*本身*。这个区分听起来微妙，但带来一个实际限制：像 `[[nodiscard]]`、`[[noreturn]]`、`[[deprecated]]` 这类常见属性，标准规定它们只能附着在函数声明（包括 lambda 的调用运算符）上，不能附着在函数*类型*上。

结果是，在 C++23 之前，下面这种写法是不合法（或者说语义对不上）的：

```cpp
auto f = [](int x) [[nodiscard]] { return x * 2; };  // 在 C++20 下编译器会警告
                                                       // "nodiscard 不能用在类型上"
```

这正是我刚才测试时复现出的错误——这个属性位置只能接受作用于函数类型的属性，而 `[[nodiscard]]`/`[[noreturn]]`/`[[deprecated]]` 恰恰不是这一类。

## C++23 怎么修的

P2173 在 lambda 语法里新增了第二个属性位置：紧跟在捕获列表之后、参数列表之前。这个新位置写出的属性，归属规则完全不同——它直接附着在调用运算符*本身*上，跟你给一个普通命名函数写 `[[nodiscard]]` 是一回事：

```cpp
auto f1 = [] [[nodiscard]] () { return 42; };   // 正确：附着在调用运算符上
auto f3 = [] [[nodiscard]] { return 99; };      // C++23 还允许省略空参数列表的圆括号
```

实测警告确认了语义生效：

```cpp
f1();   // warning: ignoring return value of 'main()::<lambda()>',
        //          declared with attribute 'nodiscard' [-Wunused-result]
```

`[[noreturn]]` 同理：

```cpp
auto f4 = [] [[noreturn]] () -> void {
    throw std::runtime_error("boom");
};
```

## 两个位置的区分要记清楚

```cpp
[capture] [[属性A]] (parameters) [[属性B]] { body }
           ^^^^^^^^               ^^^^^^^^
        附着在调用运算符本身       附着在调用运算符的类型上
        （C++23 新增位置）         （C++20 起就有的旧位置）
```

简单记法：写在参数列表*之前*的属性，对应的是"这个函数（运算符）是什么样的"——能不能丢弃返回值、是否永不返回、是否已废弃；写在参数列表*之后*的属性，对应的是"这个函数类型本身"，适用范围窄得多，目前标准里没有几个标准属性真正适用于这个位置。

## 实际价值

这个特性本身没有运行时影响，纯粹是让 lambda 能享受到此前只有具名函数才能用的诊断能力：

```cpp
auto validate = [] [[nodiscard]] (int code) {
    return code == 0;
};

validate(get_error_code());   // 漏写 if 判断会被编译器警告——
                               // 这正是 nodiscard 在普通函数上的价值，
                               // 现在 lambda 也能拥有
```

在大量依赖 lambda 写校验函数、错误码检查、状态判断的代码里，这能补上一类此前漏不到的编译期检查：调用方忘了处理返回值，编译器会主动提醒，而不是依赖人工 review。