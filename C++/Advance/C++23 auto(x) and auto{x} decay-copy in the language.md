#### auto(x) and auto{x} decay-copy in the language

## 这个特性解决的问题

C++ 里经常需要"拿到一个去掉引用、去掉 const/volatile、数组退化为指针之后的纯值类型副本"。这个操作叫 decay-copy，标准库内部到处都在用——比如 `std::thread` 构造函数把参数存进内部状态前，`std::make_pair`、`std::any` 构造时，都依赖这个语义。

但在 C++23 之前，语言本身没有提供任何直接写出这个动作的语法。唯一的办法是绕一圈：

```cpp
template<typename T>
auto decay_copy(T&& x) {
    return std::forward<T>(x);   // 利用函数返回值类型推导触发 decay
}

auto y = decay_copy(ref);
```

这要么得手写一个辅助函数模板，要么得依赖某种间接技巧，没法在表达式内联地完成这个意图。C++23 把这个能力直接做进了语言：`auto(x)` 和 `auto{x}` 这两种写法，本质就是"functional-style cast 到 auto 推导出的类型"，效果等价于把 `x` 做了一次 decay-copy。

## 具体做了什么

`auto(x)` 会按照模板参数推导（具体是 `auto` 类型推导，规则和函数模板参数推导一致）的方式，把 `x` 的类型做以下处理后产生一个新对象：

- 去掉引用（无论是左值引用还是右值引用）
- 去掉顶层 cv 限定符（`const`、`volatile`）
- 数组类型退化为指向首元素的指针
- 函数类型退化为函数指针

```cpp
int x = 10;
int& ref = x;

auto y = auto(ref);   // decltype(y) 是 int，不是 int&
auto z = auto{ref};   // 大括号写法，效果相同

int arr[3] = {1, 2, 3};
auto p = auto(arr);   // decltype(p) 是 int*，数组退化为指针

auto& r = get_ref();
auto val = auto(r);   // 哪怕 r 是引用，val 也是一份真正的拷贝
```

`auto(x)` 和 `auto{x}` 在语义上完全等价，区别仅在于一个用圆括号、一个用大括号，跟普通的函数式转型 `T(x)` 和列表初始化 `T{x}` 的关系一致——大括号形式会拒绝缩窄转换（narrowing conversion），圆括号形式不会。

## 典型用途

最直接的用途是在范围 for 循环或泛型代码里，明确表达"我要的是一份独立副本，不是引用"：

```cpp
for (auto x : auto(get_some_proxy_reference())) {
    // 强制脱离任何隐藏的引用语义（比如 proxy reference、bitfield reference）
}
```

另一个常见场景是在 lambda 捕获里，确保捕获的是值而不是引用，尤其是当源表达式本身的类型不那么直观时：

```cpp
auto lam = [val = auto(some_expression)]() { /* ... */ };
```

以及在通用代码里替代手写的 `decay_copy` 辅助函数，作为标准化、随处可用的写法。

## 为什么要进语言层面而不是留在库里

这个提案的核心动机是：decay-copy 这个语义已经隐含在 `auto` 类型推导规则里了，只是过去没有任何语法能直接"触发"这条规则而不需要一个真实的变量声明或函数调用。`auto(x)` 把这条已有规则暴露成了一个表达式形式的操作符，不需要额外分配函数调用开销，也不需要模板实例化，纯粹是编译期的类型转换语义，运行时就是一次普通的拷贝/移动构造。