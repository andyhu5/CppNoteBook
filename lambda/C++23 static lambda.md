## C++23 中的 static operator[]、static operator() 和 static lambda

### 改变了什么

在 C++23 之前，`operator()` 和 `operator[]` 必须是非静态成员函数，即便调用过程根本不需要访问对象状态也不例外。C++23（`operator()` 与 static lambda 对应 P1169R4，`operator[]` 对应 P2589R1）解除了这个限制，允许在类没有任何数据需要操作时，把这两个运算符声明为 `static`。

### 这解决了什么问题

非静态成员函数永远携带一个隐式对象参数——也就是 `this`——即便函数体从未用到它。对于一个无状态的函数对象来说，这个指针纯粹是开销：它占据一个寄存器（或栈位置）传进函数，函数却根本不会读它。

```cpp
struct IsEven {
    bool operator()(int n) const { return n % 2 == 0; }   // C++23 之前
};
```

在 x86-64 的 System V ABI 下，第一个整数参数会落在 `rdi` 寄存器里。对于非静态成员函数，`this` 占用了 `rdi`，真正的参数 `n` 被挤到了 `rsi`。即便 `this` 从未被读取，生成的代码依然带着这层间接性：

```
IsEven::operator()(const IsEven*, int):
    mov eax, esi      ; 参数落在了第二个寄存器
    and eax, 1
    xor eax, 1
    ret
```

用上 `static operator()` 之后，根本没有隐式对象参数，参数直接进入 `rdi`：

```cpp
struct IsEven {
    static bool operator()(int n) { return n % 2 == 0; }   // C++23
};
IsEven::operator()(int):
    mov eax, edi
    and eax, 1
    xor eax, 1
    ret
```

单次调用的差异很小，但标准库对函数对象的实例化非常密集（比较器、投影、range 适配器都在用），所以在大量依赖函数对象的代码库里，省下的寄存器和消除的间接调用累积起来是有意义的。

### `operator[]`

下标运算符过去也有同样的限制，现在同样得到了修复。这一点在 C++23 多维 `operator[]`（也就是逗号下标特性，`mdspan` 正是用的这个）场景下尤其有价值——一个无状态的查表类型或下标计算类型，能享受到同样的零开销待遇：

```cpp
struct LookupTable {
    static int operator[](int i) { return i * i; }
};

LookupTable t;
t[5];               // 25 —— 可以在实例上调用，但不会生成隐式对象参数
LookupTable{}[5];   // 同样合法，因为函数体内根本没有读取这个临时对象，也就没有生命周期问题
```

### Static lambda

lambda 的调用运算符默认是非静态的（如果没写 `mutable`，还会是 `const`）。这个提案给 lambda 引入了一个新的说明符，让你可以在 lambda 声明符里显式选用同样的特性：

```cpp
auto is_even = [](int n) static { return n % 2 == 0; };
```

这是显式选用而非自动生效，因为委员会希望行为是可预测的，而不是编译器悄悄根据"是否有捕获"来改变调用运算符的性质。由此引出两条直接限制：

带捕获的 lambda 不能声明为 `static`，因为静态成员函数没有 `this`，捕获的变量也就无处可依：

```cpp
int x = 5;
auto bad = [x](int y) static { return x + y; };
// 错误：'static' lambda specifier with lambda capture
```

`mutable` 的 lambda 也不能同时是 `static`，因为 `mutable` 的意义本来就是相对于隐式对象而言的——它放宽了 `this` 的常量性。没有了 `this`，这两个说明符自然互相矛盾：

```cpp
auto bad = [](int y) mutable static { return y; };
// 错误：'static' specifier conflicts with 'mutable'
```

### 什么时候该用

这纯粹是面向热路径的微优化，不带任何语义上的影响。值得用它的场景是真正无状态的函数对象、比较器、投影函数，或者会在紧密循环中反复调用、或被泛型代码（比如 range 适配器）反复实例化的 lambda。对于普通业务代码，这点差异基本无法察觉，到处都写 `static` 带来的冗余反而不值得——而且小型非静态运算符往往会被优化器内联，内联之后隐式对象参数也就一并消失了。这个收益真正变得实际，主要是在内联没有发生的情况下：函数体太大、通过函数指针间接调用，或者被 `std::function` 这类类型擦除包装起来的场景。

```c
#include <iostream>
int main(int argc, char const *argv[])
{
    auto is_even =  [](int n) mutable static { return n % 2 == 0; };
    // error: ‘static’ specifier conflicts with ‘mutable’
    //change std::cout output to bool type
    std::cout << std::boolalpha;
    std::cout << is_even(4) << "\n";
    return 0;
}
```

