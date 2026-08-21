# C++23：`Improve default container formatting`（P2585）解释

这是 Barry Revzin 提出的提案（P2585，2022），用来**完善** C++23 已有的"格式化 range/容器"能力（即 P2286 引入的 `std::format` 对容器的支持）。它解决的核心问题是：**默认的容器格式化输出，如何根据容器"像什么"来自动选择合适的展示形式**（列表形式 `[...]`、字符串形式，还是 map 形式 `{k: v, ...}`）。

## 1. 背景：P2286 已经做了什么

C++23 的 P2286 让满足 `ranges::range` 概念的类型可以直接被 `std::format` 格式化，默认输出形如：

```cpp
std::vector<int> v = {1, 2, 3};
std::format("{}", v);  // "[1, 2, 3]"

std::map<std::string, int> m = {{"a", 1}, {"b", 2}};
std::format("{}", m);  // "{"a": 1, "b": 2}"
```

但这里有个问题：**P2286 判断"是不是 map"是通过检查该类型的名字是不是标准库预定义的 `std::map` / `std::unordered_map` 等**，这种硬编码方式不通用。

## 2. P2585 改进的核心问题

如果你自己写了一个类 map 容器（比如第三方库的 `flat_map`，或者你自己实现的关联容器），它**不是**标准库那几个类型名，按 P2286 的规则就不会被识别成"map"，只能退化成普通 range，输出像这样：

```cpp
my::flat_map<std::string, int> fm = {{"a", 1}};
// P2286 之前：格式化成 [("a", 1)]  —— 不理想，看起来像 pair 的列表
// 而不是期望的 {"a": 1}
```

P2585 提出：**不要靠类型名字（nominal typing），而要靠"结构特征"（duck typing）来判断容器种类**。具体是通过检测：

1. 该 range 的 `reference` 类型是否是一个"二元组"（`std::pair` 或 size 为 2 的 `std::tuple`）
2. 该类型是否同时定义了 `key_type` 和 `mapped_type`（这是 map 的标志性 typedef）

如果两者都满足，就自动按 **map 格式**（`{key: value, ...}`）格式化；如果只是"元素是 pair 但没有 key_type/mapped_type"，就按普通 **set/序列格式**（`[(a, b), ...]`）处理。

```cpp
// 只要满足结构特征，就能被正确识别为 map，无需是标准库类型
namespace my {
    template <class K, class V>
    struct flat_map {
        using key_type = K;
        using mapped_type = V;
        // ... range 接口，reference 类型是 pair<const K, V>
    };
}

my::flat_map<std::string, int> fm = {{"a", 1}, {"b", 2}};
std::format("{}", fm);  // 现在能输出 {"a": 1, "b": 2}
```

## 3. 为什么这样设计是安全的

论文里特别讨论了"会不会误判"的问题：

- 检查条件很严格：**同时**要求 `key_type` + `mapped_type` typedef **加上** reference 是二元组，这样几乎不可能有"看起来像 map 但其实不是"的意外类型被误判。
- 即使真的判断错了，**最坏情况也只是格式选择不理想**（比如把不该当 map 的东西格式化成 map 的样子），元素本身仍然会被完整格式化出来，不会造成编译错误或者信息丢失。
- 提案特意**没有**去处理"这个容器是不是该被当成字符串"的猜测（比如检测是否能转成 `std::string`/`string_view`/`char const*`），因为这种猜测更容易出错、更难界定边界，所以刻意留给用户自己特化 formatter 处理。

## 4. 影响范围（措辞层面）

从被检索到的提案文本可以看到，标准措辞把原本"专门给 `map`/`multimap`/`unordered_map`/`unordered_multimap` 这几个具名类型写的 formatter 特化"，改成了一个更通用的 `default-range-formatter<range_format_kind::map, R, charT>` 模板，用 range 的属性（`R`）而不是具体类型名去参数化。这意味着标准库内部实现也统一成了"基于结构特征分派"的机制，而不是分别为每个具名容器写一份特化。

## 5. 小结

|                      | P2286（之前）                                | + P2585（改进后）                                       |
| -------------------- | -------------------------------------------- | ------------------------------------------------------- |
| 判断"是不是 map"     | 硬编码检查是否是 `std::map` 等标准库具名类型 | 检查结构特征：`key_type`+`mapped_type`+二元组 reference |
| 自定义 map-like 容器 | 默认格式化成序列 `[(a,b),...]`               | 自动识别为 map，格式化成 `{a: b, ...}`                  |
| 安全性               | —                                            | 严格双重检查，误判代价小（仍能格式化，只是形式不理想）  |

测试：

```c
#include <unordered_map>
#include <vector>
#include <print>

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
}
```

```
//output:

map: {3: "three", 2: "two", 1: "one"}
vector: [1, 2, 3, 4, 5]
vector of pairs: [(1, "one"), (2, "two"), (3, "three")]
vector of maps: [{1: "one"}, {2: "two"}, {3: "three"}, {4: "four"}]
```

