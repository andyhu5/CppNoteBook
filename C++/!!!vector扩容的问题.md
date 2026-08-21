```c++
#include <iostream>
#include <vector>
#include <ranges>

struct MyType {
    int value;
    MyType(int v) : value(v) { std::cout << "Constructed " << value << "\n"; }
    MyType(const MyType&) { std::cout << "Copied\n"; }
    MyType(MyType&&) noexcept { std::cout << "Moved\n"; }
};

int main() {
    std::vector<MyType> vec{2,4};

    // Using as_rvalue to treat elements as rvalues
    auto view = vec | std::views::as_rvalue;

    // Move elements into a new vector
    std::vector<MyType> new_vec;
    new_vec.reserve(2); // Reserve space to avoid reallocations ，如果不reserver会导致vector扩容，重新调用move拷贝构造；
    for (auto&& item : view) {
        new_vec.emplace_back(std::move(item));
    }
}

```

***为什么输出打印有3个Moved ？ ***

这是一个典型的 **`std::vector` 扩容（reallocation）** 导致的现象，与 `std::views::as_rvalue` 本身关系不大。

我们一步一步分析。

------

## 第一步：构造 `vec`

```cpp
std::vector<MyType> vec{2,4};
```

发生：

```
Constructed 2
Constructed 4
```

接下来：

```
Copied
Copied
```

为什么有两个 `Copied`？

因为：

```cpp
std::vector<MyType> vec{2,4};
```

实际上等价于：

```cpp
std::vector<MyType> vec{
    MyType(2),
    MyType(4)
};
```

`initializer_list` 中的元素是 **const** 的，因此 `vector` **不能移动**，只能复制：

```
initializer_list
        │
        ▼
const MyType
        │
        ▼
vector
```

所以得到：

```
Copied
Copied
```

------

## 第二步：遍历 `as_rvalue`

```cpp
auto view = vec | std::views::as_rvalue;
```

这里：

```cpp
decltype(*view.begin())
```

已经变成：

```cpp
MyType&&
```

因此：

```cpp
new_vec.emplace_back(std::move(item));
```

第一次：

```
Moved
```

第二次：

理论上也是

```
Moved
```

为什么最后变成 **3 次**？

------

## 第三步：`new_vec` 扩容

你的代码：

```cpp
std::vector<MyType> new_vec;
```

开始时：

```
size = 0
capacity = 0
```

第一次：

```cpp
emplace_back(...)
```

通常：

```
capacity

0

↓

1
```

放进去一个元素：

```
Moved
```

目前：

```
size = 1
capacity = 1
```

------

第二次：

```cpp
emplace_back(...)
```

容量不足：

```
size = 1
capacity = 1
```

需要扩容：

```
capacity

1

↓

2
```

扩容时：

旧元素必须搬到新的内存：

```
旧内存

element0

↓

新内存
```

由于你定义了：

```cpp
MyType(MyType&&)
```

因此使用***移动构造***：

```
Moved
```

然后：

第二个元素再构造：

```
Moved
```

所以第二次 `emplace_back` 实际发生：

```
Moved   // 搬运旧元素
Moved   // 插入新元素
```

------

## 总计

所以：

```
Constructed 2
Constructed 4
Copied
Copied

Moved        // 第一个 emplace_back

Moved        // vector 扩容搬运旧元素
Moved        // 第二个 emplace_back
```

正好：

```
3 个 Moved
```

------

## 如何验证？

提前：

```cpp
new_vec.reserve(2);
```

改成：

```cpp
std::vector<MyType> new_vec;
new_vec.reserve(2);

for (auto&& item : view)
{
    new_vec.emplace_back(std::move(item));
}
```

输出会变成：

```
Constructed 2
Constructed 4
Copied
Copied

Moved
Moved
```

只有两个 `Moved`。

因为：

```
capacity = 2
```

无需扩容。

------

## 更明显的实验

例如：

```cpp
new_vec.reserve(100);
```

无论放多少元素，

都不会因为扩容而产生额外：

```
Moved
```

------

## 与 `views::as_rvalue` 有没有关系？

没有。

即使不用：

```cpp
views::as_rvalue
```

写：

```cpp
new_vec.emplace_back(std::move(item));
```

也会看到同样的扩容移动。

`as_rvalue` 的作用只是：

```cpp
MyType&

↓

MyType&&
```

让元素以右值引用的形式暴露出来，便于移动。

真正多出来的那个 `Moved` 是：

> **`std::vector` 在扩容时，为了重新分配内存而移动已有元素。**

------

### 如果想观察每一次移动的是哪个对象

建议把移动构造函数改成：

```cpp
MyType(MyType&& other) noexcept : value(other.value)
{
    std::cout << "Moved " << value << '\n';
}
```

以及复制构造函数改成：

```cpp
MyType(const MyType& other) : value(other.value)
{
    std::cout << "Copied " << value << '\n';
}
```

这样你会看到类似输出：

```text
Constructed 2
Constructed 4
Copied 2
Copied 4
Moved 2      // 第一次插入
Moved 2      // 扩容时搬迁旧元素
Moved 4      // 第二次插入
```

这能直观地区分**移动已有元素**和**移动新插入元素**。

