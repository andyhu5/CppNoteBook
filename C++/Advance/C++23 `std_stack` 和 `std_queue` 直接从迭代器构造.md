C++23 之前，`std::stack` 和 `std::queue`（以及 `std::priority_queue`）有一个长期被吐槽的"缺陷"：**它们的容器适配器构造函数不支持直接从一对迭代器构造**，你只能先构造一个底层容器，再用它去初始化适配器。这个改动（提案 **P1425R4**）就是为了填补这个空缺。

## 问题背景

在 C++23 之前：

```cpp
#include <stack>
#include <queue>
#include <vector>
#include <deque>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};

    // 错误！不能直接用迭代器对构造
    // std::stack<int> s(v.begin(), v.end());       // 编译失败
    // std::queue<int> q(v.begin(), v.end());       // 编译失败

    // 只能这样绕一圈：先构造底层容器，再传给适配器
    std::stack<int> s(std::deque<int>(v.begin(), v.end()));
    std::queue<int> q(std::deque<int>(v.begin(), v.end()));
}
```

这很不方便，尤其是当你只是想"拿一段数据初始化一个 stack/queue"时，却要多写一层中间容器的构造。

## C++23 的改动

新增了接受一对迭代器的构造函数：

```cpp
namespace std {
    template<class Container>
    class stack {
    public:
        // 新增（简化签名）
        template<class InputIterator>
        stack(InputIterator first, InputIterator last);

        template<class InputIterator, class Alloc>
        stack(InputIterator first, InputIterator last, const Alloc& alloc);
        // ...
    };

    template<class Container>
    class queue {
    public:
        template<class InputIterator>
        queue(InputIterator first, InputIterator last);

        template<class InputIterator, class Alloc>
        queue(InputIterator first, InputIterator last, const Alloc& alloc);
        // ...
    };

    // priority_queue 同理
}
```

## 使用示例

```cpp
#include <stack>
#include <queue>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};

    // C++23 起可以直接这样写
    std::stack<int> s(v.begin(), v.end());
    std::queue<int> q(v.begin(), v.end());

    std::cout << s.top() << '\n';   // 5（stack 底层默认是 deque，插入顺序即遍历顺序，栈顶是最后插入的元素）
    std::cout << q.front() << '\n'; // 1（queue 的队首是最先插入的元素）
}
```

## 与容器本身迭代器构造的一致性

这个改动的意义在于让 `stack`/`queue`/`priority_queue` 的构造方式和标准容器（`vector`、`deque`、`list` 等）保持**一致的接口习惯**——你已经习惯了 `std::vector<int> v(first, last)`，现在容器适配器也能这样用了，减少了认知负担和样板代码。

## priority_queue 的额外重载

`priority_queue` 因为涉及比较器，还额外提供了带比较器的版本：

```cpp
namespace std {
    template<class T, class Container, class Compare>
    class priority_queue {
    public:
        template<class InputIterator>
        priority_queue(InputIterator first, InputIterator last,
                        const Compare& compare = Compare());

        template<class InputIterator>
        priority_queue(InputIterator first, InputIterator last,
                        const Compare& compare, const Container& cont);

        template<class InputIterator>
        priority_queue(InputIterator first, InputIterator last,
                        const Compare& compare, Container&& cont);
        // ...
    };
}
```

示例：

```cpp
std::vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};

// 直接用迭代器对 + 自定义比较器构造小顶堆
std::priority_queue<int, std::vector<int>, std::greater<int>> pq(
    v.begin(), v.end(), std::greater<int>());

std::cout << pq.top() << '\n'; // 1（最小值在顶）
```

## 是否支持推导 CTAD？

配合类模板推导指引（CTAD），甚至可以省略模板参数：

```cpp
std::vector<int> v = {1, 2, 3};
std::stack s(v.begin(), v.end());   // 自动推导为 stack<int>
std::queue q(v.begin(), v.end());   // 自动推导为 queue<int>
```

（具体推导行为依赖标准库实现是否已完整支持对应的推导指引，需注意不同编译器/标准库版本的支持进度。）

## 小结

| 特性                                 | C++20 及之前                     | C++23  |
| ------------------------------------ | -------------------------------- | ------ |
| 迭代器对直接构造 stack/queue         | ❌ 不支持，需先构造底层容器再传入 | ✅ 支持 |
| priority_queue 迭代器对 + 比较器构造 | ❌ 不支持                         | ✅ 支持 |
| 与其他标准容器构造习惯一致性         | 不一致                           | 一致   |

这是一个典型的"查漏补缺"型小改进——不涉及新概念，纯粹是接口易用性上的提升，减少了不必要的样板代码。