# C++23：`Formatting Ranges`（P2286）解释

这是 Barry Revzin 的提案（P2286），是 C++23 里让 `std::format`/`std::print` 能够**直接格式化 range（容器/范围）** 的核心提案。它是我们前两次讨论的 `thread::id`/`stacktrace` 格式化以及 `Improve default container formatting`（P2585）这些"配套改进"的**基础**——没有 P2286，就没有后面那些完善提案。

## 1. 解决的问题

C++20 之前，如果你想打印一个 `vector`，只能自己手写循环：

```cpp
std::vector<int> v = {1, 2, 3};

// 之前只能这样
std::cout << "[";
for (size_t i = 0; i < v.size(); ++i) {
    if (i) std::cout << ", ";
    std::cout << v[i];
}
std::cout << "]" << std::endl;
```

C++23 之后，只要元素本身可格式化，容器本身也自动可格式化：

```cpp
#include <vector>
#include <format>

std::vector<int> v = {1, 2, 3};
std::format("{}", v);   // "[1, 2, 3]"
```

## 2. 支持的范围与默认输出形式

P2286 根据容器的"种类"（`range_format`）自动选择合适的括号/分隔符：

```cpp
std::vector<int> v = {1, 2, 3};
std::format("{}", v);              // [1, 2, 3]

std::set<int> s = {1, 2, 3};
std::format("{}", s);              // {1, 2, 3}   （集合用花括号）

std::map<std::string, int> m = {{"a", 1}, {"b", 2}};
std::format("{}", m);              // {"a": 1, "b": 2}   （map 用 key: value）

std::pair<int, std::string> p = {1, "x"};
std::format("{}", p);              // (1, "x")

std::tuple<int, double, char> t = {1, 2.5, 'c'};
std::format("{}", t);              // (1, 2.5, 'c')

std::vector<std::vector<int>> vv = {{1, 2}, {3, 4}};
std::format("{}", vv);             // [[1, 2], [3, 4]]  嵌套也自动支持
```

字符串元素会自动加引号（因为它调用的是元素类型的 `formatter`，而 `string`/`char` 的默认 formatter 就是加引号/转义处理，这也是 P2286 顺带规定的行为）。

## 3. 格式说明符的支持

P2286 不只是"能打印"，还允许在 `{}` 里加格式说明符控制细节：

```cpp
std::vector<int> v = {1, 2, 3};

std::format("{}", v);      // [1, 2, 3]
std::format("{:n}", v);    // 1, 2, 3        —— n：去掉外层括号
std::format("{::#x}", v);  // [0x1, 0x2, 0x3] —— 冒号后面的部分应用给"每个元素"

// 控制宽度、填充也支持
std::format("{:*^20}", v); // 用 * 填充居中对齐，总宽20
```

语法上，`:` 之后第一段是控制容器本身（宽度、对齐、`n` 去括号），再一个 `:` 之后的部分会转发给**每个元素的 formatter**，这样可以精细控制元素怎么打印（例如上面的 `#x` 是给每个 `int` 元素用十六进制并带前缀）。

## 4. 底层机制：`range_formatter` 与 `formattable`

标准库为此新增了：

- `std::range_formatter<T, charT>`：可复用的通用组件，如果你自己写了一个 range 类型，也可以在自己的 `formatter` 特化里内部委托给 `range_formatter` 来复用这套逻辑，不用自己重新实现分隔符/括号/元素格式化的处理。
- `std::formattable<T, charT>` concept：判断某类型是否可以用 `std::format` 格式化，range formatter 内部就是靠这个 concept 递归判断"元素是否也可格式化"。

```cpp
// 自定义类型想复用 range 的格式化逻辑
template <typename T>
struct std::formatter<my::container<T>> 
    : std::range_formatter<T> {
    // 可以选择性覆盖某些行为
};
```

## 5. 需要注意的边界情况

- **`std::string` / `std::string_view` 本身也是 range**（`range<char>`），但它们被显式排除在这套"容器格式化"规则外，仍然按普通字符串处理，不会被拆成 `['h','e','l','l','o']` 这种荒谬的输出。
- 是否被当作"容器"来格式化，取决于该类型是否满足 `ranges::input_range` 且元素可格式化；如果元素本身不可格式化（比如元素类型没有对应的 `formatter` 特化），那整个容器也不可格式化，会在编译期报错，而不是运行时崩溃。
- 这套机制默认只覆盖"标准定义的几种容器分类"（sequence / set / map），对于用户自定义容器"该被当成 set 还是 map"的**精确判断规则**，后续被 P2585（我们上一轮讨论的提案）进一步完善，改成了基于结构特征而非硬编码类型名。

## 6. 一句话总结

**P2286 让所有满足 range 概念、且元素本身可格式化的容器（vector/set/map/pair/tuple及其嵌套）都能被 `std::format`/`std::print` 直接输出，还提供了统一的格式说明符语法来控制容器整体样式和逐元素样式**——这是 C++23 格式化生态里最基础也是最常用的一块拼图。



```c
#include <unordered_map>
#include <vector>
#include <print>
#include <set>
#include <map>

int main() {
    std::unordered_map<int, std::string> my_map = {
        {1, "one"},
        {2, "two"},
        {3, "three"}
    };
    std::print("map: {}\n", my_map); // 输出: map: one

    std::vector<int> my_vector = {1, 2, 3, 4, 5};
    std::print("vector: {}\n", my_vector); // 输出: vector:

    std::vector<std::pair<int, std::string>> my_vector_of_pairs = {
        {1, "one"},
        {2, "two"},
        {3, "three"}
    };
    std::print("vector of pairs: {}\n", my_vector_of_pairs); // 输出

    std::vector<std::unordered_map<int, std::string>> my_vector_of_maps = {
        {{1, "one"}}, {{2, "two"}},
        {{3, "three"}}, {{4, "four"}}
    };
    std::print("vector of maps: {}\n", my_vector_of_maps); // 输出

    std::vector<int> v = {1, 2, 3};
    std::print("vector: {}\n", std::format("{}", v)); // [1, 2, 3]

    std::set<int> s = {1, 2, 3};
    std::print("set: {}\n", std::format("{}", s)); // {1, 2, 3}   （集合用花括号）

    std::map<std::string, int> m = {{"a", 1}, {"b", 2}};
    std::print("map: {}\n", std::format("{}", m)); // {"a": 1, "b": 2}   （map 用 key: value）

    std::pair<int, std::string> p = {1, "x"};
    std::print("pair: {}\n", std::format("{}", p)); // (1, "x")

    std::tuple<int, double, char> t = {1, 2.5, 'c'};
    std::print("tuple: {}\n", std::format("{}", t)); // (1, 2.5, 'c')

    std::vector<std::vector<int>> vv = {{1, 2}, {3, 4}};
    std::print("vector of vectors: {}\n", std::format("{}", vv)); // [[1, 2], [3, 4]]  嵌套也自动支持
}
```

```
## output: ##

map: {3: "three", 2: "two", 1: "one"}
vector: [1, 2, 3, 4, 5]
vector of pairs: [(1, "one"), (2, "two"), (3, "three")]
vector of maps: [{1: "one"}, {2: "two"}, {3: "three"}, {4: "four"}]
vector: [1, 2, 3]
set: {1, 2, 3}
map: {"a": 1, "b": 2}
pair: (1, "x")
tuple: (1, 2.5, 'c')
vector of vectors: [[1, 2], [3, 4]]
```

