

#### C++20 的 `constexpr std::string`  constexpr std::vector

```c++

#include <string>
#include <vector>

constexpr std::string getMsg() {
    return "Hello World";
}

constexpr bool checkMsg() {
    static_assert(getMsg().size() == 11, "Message size is not 11");
    return true;
}

constexpr std::vector<int> getData() {
    return {1, 2, 3};
}
constexpr bool check() {
  auto s = getMsg();
  return s.size() == 11;
}

int main(int argc, char const *argv[])
{
    static_assert(getData().size() == 3);

    static_assert(checkMsg());
    static_assert(check());

    for (auto &&i : getData())
    {
        std::cout << i << std::endl;
    }

    return 0;
}

```

**P0426R1 的角色**是 C++17 里的一块地基——它回应了标准库缺陷报告 LWG2232（要求 `char_traits` 的 `length()`、`compare()`、`find()` 声明为 constexpr），让字符操作可以在编译期执行。这直接使 `std::string_view` 的大部分操作在 C++17 就能 constexpr 化。

**C++20 的 `constexpr std::string`/`std::vector`** 则更进一步，依赖了另一个关键特性——编译期 `new`/`delete`。

这带来了一个微妙的限制：

- `constexpr` **函数**里使用 `std::string`/`std::vector` ✓（局部分配，局部释放）
- `constexpr` **变量**存储 `std::string`/`std::vector` ✗（堆内存会"逃逸"到运行时）

你代码中的 `constexpr auto result = getMsg()` 属于第二种情况，编译器会报错。解决方案就是上面图中展示的两条路：

用 `static_assert` 在编译期消费掉结果，

或改用 `std::string_view` / `std::array` 这类不依赖***堆分配的类型***。