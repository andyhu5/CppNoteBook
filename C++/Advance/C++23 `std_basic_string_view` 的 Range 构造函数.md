# `std::basic_string_view` 的 Range 构造函数

## 背景：C++17 `string_view` 原有的构造方式

C++17 引入 `std::string_view` 时，只提供了几种有限的构造方式：

```cpp
constexpr basic_string_view() noexcept;                          // 默认构造，空视图
constexpr basic_string_view(const CharT* s);                     // 从 C 字符串（走 strlen）
constexpr basic_string_view(const CharT* s, size_type count);    // 指针 + 长度
constexpr basic_string_view(const basic_string_view&) noexcept;  // 拷贝构造
```

这意味着，如果你有一个**自定义的、连续存储字符的容器**（不是 `std::string`，比如自己写的 `SmallCharBuffer`，或者第三方库的字符数组包装类），想把它转换成 `string_view`，只能手动拆解成指针+长度：

```cpp
MyCharBuffer buf = ...;
std::string_view sv(buf.data(), buf.size()); // 每次都要手动拆
```

## C++20：新增 Range 构造函数

C++20 给 `basic_string_view` 增加了一个**通用的 range 构造函数**，只要传入的对象满足特定约束（本质上是"一段连续存储的、字符类型兼容的序列"），就能直接构造 `string_view`，不需要手动拆成指针+长度：

```cpp
template<class R>
constexpr basic_string_view(R&& r);
```

约束大致要求 `R`：

- 是一个 `contiguous_range`（连续存储的 range，如 `std::array`、`std::vector<char>`、C 数组等，但**不能**是 `std::basic_string_view` 自身或 `std::basic_string` 相关的特殊情形，避免和已有构造函数冲突）；
- `std::ranges::range_value_t<R>` 与 `CharT` 兼容（类型相同）；
- 不能是字符指针类型（否则和 `const CharT*` 构造函数冲突）；
- 具备合适的 `data()`/`size()` 语义。

```cpp
std::vector<char> v = {'h', 'e', 'l', 'l', 'o'};
std::string_view sv(v); // C++20 起：直接从 vector<char> 构造，无需手动拆 data()/size()

std::array<char, 5> arr = {'w', 'o', 'r', 'l', 'd'};
std::string_view sv2(arr); // 同样直接可用
```

这大大提升了 `string_view` 和各种"连续字符容器"之间的互操作性，尤其对**泛型代码**（模板函数接受任意满足要求的字符容器，统一转成 `string_view` 处理）非常友好。

## C++23：把这个构造函数改为 `explicit`

问题在于，C++20 里这个 range 构造函数是**隐式**的，这带来了两个隐患：

### 隐患 1：意外的隐式转换，可能引发生命周期问题

```cpp
std::string_view get_view() {
    std::vector<char> local = {'d', 'a', 't', 'a'};
    return local; // ❌ C++20：隐式转换合法，编译通过，但 local 在函数返回后被销毁，
                   //    string_view 变成悬垂引用！这是一个极其隐蔽的 bug
}
```

因为构造函数是隐式的，`return local;` 这种写法**编译器不会有任何警告**，看起来像是返回了一个正常的值，实际上返回的 `string_view` 立刻变成悬垂引用——这是"生命周期陷阱"里最危险的一类：**编译通过、运行期才炸**。

### 隐患 2：不经意间发生的类型转换，掩盖了程序员的真实意图

```cpp
void process(std::string_view sv);

std::vector<char> buffer = load_data();
process(buffer); // 一眼看不出这里发生了"容器 → 视图"的转换，容易误以为 process 接受的是 vector 本身
```

隐式转换在这类场景下降低了代码的可读性——阅读者无法一眼看出这是一次"类型转换"还是"本来就匹配的类型"。

## C++23 的修正

C++23 把这个 range 构造函数标记为 `explicit`：

```cpp
template<class R>
constexpr explicit basic_string_view(R&& r); // C++23 起：explicit
```

修正后：

```cpp
std::vector<char> v = {'h', 'e', 'l', 'l', 'o'};

std::string_view sv1 = v;      // ❌ C++23 起：编译错误，不允许隐式转换
std::string_view sv2{v};       // ✅ OK，显式构造（直接初始化）
std::string_view sv3(v);       // ✅ OK，显式构造（函数风格直接初始化）

void process(std::string_view sv);
process(v);                    // ❌ C++23 起：编译错误！函数实参传递属于"复制初始化"语境，
                                //    explicit 构造函数在这里不能被隐式调用
process(std::string_view{v});  // ✅ OK，显式转换后传入
```

之前那个悬垂引用的例子，在 C++23 下会直接编译失败：

```cpp
std::string_view get_view() {
    std::vector<char> local = {'d', 'a', 't', 'a'};
    return local; // ✅ C++23 起：编译错误！return 语句属于复制初始化，
                   //   explicit 构造函数不能隐式参与，问题在编译期就被拦截
}
```

## 为什么值得这样"牺牲一点易用性"

`string_view` 的核心特性就是**非拥有（non-owning）**——它只是"借用"别人的内存，不管理生命周期。任何"容器 → `string_view`"的转换，本质上都隐含着一个使用者必须自己确保的契约："**原容器的生命周期必须比这个 `string_view` 更长**"。这种带有生命周期契约的转换，理应要求程序员**显式地**做出这个决定，而不是被编译器悄悄地、隐式地完成——这正是很多现代 C++ 安全实践（比如 `explicit` 关键字本身的设计哲学）反复强调的原则：**任何可能带来所有权/生命周期歧义的转换，都不应该是隐式的**。

## 小结

`basic_string_view` 的 range 构造函数是 C++20 引入的一项提升泛型互操作性的特性，让任意连续字符容器都能方便地转换为 `string_view`。但 C++23 发现这个构造函数保持**隐式**存在明显的生命周期风险（尤其是"返回局部容器的视图"这种典型悬垂引用场景），于是将其收紧为 `explicit`，用"多敲几个字符"的代价，把一类潜在的运行期悬垂引用 bug，提前拦截在编译期——这与本轮讨论过的"`nullptr` 构造被禁用"这项 `string_view` 相关改动，属于同一批**加强 `string_view` 使用安全性**的 C++23 修订。