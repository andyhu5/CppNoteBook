std::to_array 是 C++20 在 <array> 头文件中引入的工具函数，用于将 C 风格内置数组（built-in array）或初始化列表转换为 std::array。 

en.cppreference.com

1. 函数签名

cpp

```cpp
#include <array>

// (1) 左值引用版本（拷贝）
template <class T, std::size_t N>
constexpr std::array<std::remove_cv_t<T>, N> 
to_array(T (&a)[N]);

// (2) 右值引用版本（移动）
template <class T, std::size_t N>
constexpr std::array<std::remove_cv_t<T>, N> 
to_array(T (&&a)[N]);
```

- 它只支持一维数组，多维数组不支持（会编译错误）。
- 返回的 std::array 元素类型会移除 const/volatile（remove_cv_t<T>）。
- 主要用途和优势与直接使用 std::array 的类模板参数推导（CTAD）相比，std::to_array 在以下场景特别有用：

1. 显式指定元素类型，让长度自动推导（支持隐式转换）：

   cpp

   ```cpp
   auto a = std::to_array<long>({3, 4});        // OK: std::array<long, 2>
   // std::array<long>{3, 4};                   // 错误：模板参数不足
   ```

2. 复制字符串字面量（得到字符数组而非指针）：

   cpp

   ```cpp
   auto s1 = std::to_array("foo");               // std::array<char, 4> {'f','o','o','\0'}
   auto s2 = std::array{"foo"};                  // std::array<const char*, 1>
   ```

3. 创建包含不可拷贝类型的 std::array（使用移动）：

   cpp

   ```cpp
   auto arr = std::to_array({std::make_unique<int>(42)});
   ```

4. constexpr 支持：常用于编译期常量数组。

5. 完整示例

cpp

```cpp
#include <array>
#include <string_view>
#include <utility>
#include <type_traits>
#include <iostream>

int main()
{
    // 1. 从字符串字面量创建
    auto a1 = std::to_array("hello");
    static_assert(a1.size() == 6);  // 包含 '\0'

    // 2. 类型和长度自动推导
    auto a2 = std::to_array({0, 2, 1, 3});
    static_assert(std::is_same_v<decltype(a2), std::array<int, 4>>);

    // 3. 指定类型，隐式转换 + 长度推导
    auto a3 = std::to_array<long>({0, 1, 3});
    static_assert(std::is_same_v<decltype(a3), std::array<long, 3>>);

    // 4. 复杂类型
    auto a4 = std::to_array<std::pair<int, float>>(
        {{3, 0.0f}, {4, 0.1f}, {5, 0.2f}});

    // 5. constexpr + string_view
    constexpr auto names = std::to_array<std::string_view>(
        {"Alice", "Bob", "Charlie"});
    static_assert(names.size() == 3);
    static_assert(names[2] == "Charlie");

    // 多维数组不支持
    // int arr[2][3] = {{1,2,3},{4,5,6}};
    // auto bad = std::to_array(arr);  // 编译错误
}
```

4. 注意事项

- 不复制多维数组。
- 元素类型 T 需要满足 CopyConstructible（左值版本）或 MoveConstructible（右值版本）。
- 性能上会产生临时对象，但现代编译器通常能很好优化。
- 常与 std::array 的其他 C++20 特性（如 constexpr 算法）一起使用。

std::to_array 极大提升了 std::array 在现代 C++ 中的易用性，尤其适合需要固定大小、栈上分配、编译期友好数组的场景。 

en.cppreference.com

需要更多特定用例或与 std::make_array（实验特性）的对比吗？