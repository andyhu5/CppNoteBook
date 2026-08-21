

# C++23 std::bind_back  绑定最后面的参数

```C++
#include <algorithm>
#include <vector>
#include <print>

int main() {
    std::vector<int> haystack{1,2,3,4,5,1,2,3,4,5};
    std::vector<int> needle{3,4};

    auto it = std::find_end(
        haystack.begin(), haystack.end(),
        needle.begin(), needle.end());
    // (it - haystack.begin()) == 7
    std::println("(it - haystack.begin()) == {}", (it - haystack.begin()));   

    // Ranges version returns the found instance as a subrange
    auto rng = std::ranges::find_end(haystack, needle);
    // rng == {3,4}
    std::println("rng == {}", rng);

    // Both versions support custom comparator
    std::string sentence = "Word word WORD wORD";
    std::string word = "word";

    auto case_sen = std::ranges::find_end(sentence, word);
    // case_sen == "word";
    auto case_ins = std::ranges::find_end(sentence, word, [](char l, char r){
        return std::tolower(l) == std::tolower(r);
    });
    // case_ins == "wORD"

    std::println("case_sen == {}", case_sen);
    std::println("case_ins == {}", case_ins);
}
```



这段代码展示了 C++ 中 <functional> 头文件引入的 std::bind_back（C++23 特性），用于从后向前绑定函数参数。

代码通过两个示例展示了其用法：绑定 lambda 参数和简化函数调用中的默认参数。以下是逐步解释。

------

代码概览

- 使用 std::bind_back 绑定 lambda 的后三个参数，生成新函数。
- 使用 std::bind_back 为函数 do_stuff 设置不同的默认参数组合。

------

关键组件

1. **头文件**

cpp

```cpp
#include <functional>
#include <iostream>
```

- <functional>：提供 std::bind_back。
- <iostream>：用于输出。
- **std::bind_back 与 Lambda**

cpp

```cpp
auto f = [](int a, int b, int c, int d, int e, int f) {
    std::cout << a << " " << b << " " << c << " " << 
        d << " " << e << " " << f << "\n";
};
auto bound = std::bind_back(f, 1, 2, 3);
bound(4, 5, 6);
```

- **f**：

  - Lambda 函数，接受 6 个 int 参数并打印。

- **std::bind_back**：

  - 原型：bind_back(func, args...)。
  - 将 args... 绑定到 func 的最后几个参数，返回新函数对象。
  - 新函数接受剩余的前置参数。

- **bound**：

  - **绑定 f 的最后 3 个参数（d, e, f）为 1, 2, 3。**
  - 接受 3 个参数（a, b, c）。

- **bound(4, 5, 6)**：

  - 调用 f(4, 5, 6, 1, 2, 3)。

- **输出**：

  ```text
  4 5 6 1 2 3
  ```

- **std::bind_back 与默认参数**

cpp

```cpp
struct MainArg{};
void do_stuff(MainArg, int option1, int option2, int option3) {}
constexpr auto do_stuff_one_way = std::bind_back(do_stuff, 2, 1, -3);
constexpr auto do_stuff_another_way = std::bind_back(do_stuff, -7, 6, 2);
```

- **do_stuff**：
  - 函数接受 MainArg 和 3 个 int 参数，无实现。
- **do_stuff_one_way**：
  - 使用 std::bind_back 绑定 option1 = 2, option2 = 1, option3 = -3。
  - 新函数只接受 MainArg 参数。
  - 等价于调用 do_stuff(MainArg{}, 2, 1, -3)。
- **do_stuff_another_way**：
  - 绑定 option1 = -7, option2 = 6, option3 = 2。
  - 等价于调用 do_stuff(MainArg{}, -7, 6, 2)。
- **constexpr**：
  - 表示绑定在编译期完成，生成常量表达式。
- **用法**：
  - do_stuff_one_way(MainArg{}) 和 do_stuff_another_way(MainArg{}) 调用不同参数组合。
- **注释**：
  - 示例中未执行调用，仅定义。

------

为什么这样工作？

1. **std::bind_back**：
   - 与 std::bind_front 相对，从后向前绑定参数。
   - 返回的函数对象接受未绑定参数。
2. **参数绑定**：
   - 绑定值按顺序填充函数的最后几个参数。
3. **灵活性**：
   - 替代复杂的默认参数或多个 lambda 包装。
4. **constexpr**：
   - 绑定过程无副作用，可在编译期完成。

------

输出

- 仅第一部分有输出：

  ```text
  4 5 6 1 2 3
  ```

- 第二部分未执行调用，无输出。

------

使用场景

- **简化函数调用**：
  - 绑定固定参数，减少调用时的冗余。
- **默认参数替代**：
  - 为函数提供多种预设配置。
- **回调设计**：
  - 生成特定参数的函数对象。

------

总结

- std::bind_back 绑定 lambda 的后三个参数，生成新函数，打印 4 5 6 1 2 3。
- 为 do_stuff 创建两种参数组合，展示替代默认参数的用法。
- 代码突出 std::bind_back 的便捷性和灵活性。