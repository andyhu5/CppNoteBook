C++23 ***动态宽度（dynamic width）***和***动态精度***

views::repeat

string repeat



```c
#include <iostream>
#include <ranges>
#include <string>
#include <print>

int main()
{
    auto r = std::views::repeat('*',5)|std::views::common;

    //copy r to std::string
    std::string s(r.cbegin(), r.cend());
    std::println("Repeated string: {}", s);

    //use string to construct a repeat string,e.g "*****"
    std::string repeat_string(5, '&');
    for (int n = 8; n <= 16; ++n)
    {
        std::println("{:>{}}", repeat_string, n);
    }
    
}
```

C++23 支持**动态宽度（dynamic width）**，使用嵌套替换字段 `{}`：

```
#include <print>
#include <string>

int main()
{
    std::string repeat_string(5, '&');
    int n = 8;

    std::println("{:>{}}", repeat_string, n);
}
```

输出：

```
   &&&&&
```

解释：

```
"{:>{}}"
```

表示：

- `>`：右对齐
- `{}`：宽度由下一个参数提供

参数对应关系：

```
std::println("{:>{}}",
             repeat_string,  // 第一个 {}
             n);             // 第二个 {}，作为宽度
```

## 总结

动态宽度和动态精度都使用额外的 `{}` 来提供参数：

```
std::println("{:>{}}", value, width);     // 动态宽度
std::println("{:.{}}", pi, precision);    // 动态精度
std::println("{:>{}.{}}", pi, width, precision); // 宽度+精度
```

因此，你的代码应改为：

```
std::string repeat_string(5, '&');
int n = 8;

std::println("{:>{}}", repeat_string, n);
```

这是 C++20/23 `std::format` 和 C++23 `std::print` 系列支持动态格式参数的标准写法。