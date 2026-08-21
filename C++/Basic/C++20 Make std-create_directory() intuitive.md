C++20 DR17（实际为 P1164R1） 全称是 “Make std::create_directory() intuitive”，这是一个针对 <filesystem> 库的缺陷报告（Defect Report），在 C++20 中被采纳，并回溯应用到 C++17。 

en.cppreference.com

问题背景（C++17 原始行为）在 C++17 最初发布的 std::filesystem::create_directory 中，存在一个反直觉的行为：

- 如果目标路径 p 已经存在一个非目录的文件（比如一个普通文件、符号链接等），函数不会报告错误，而是安静地返回 false。
- 这与大多数开发者的直觉不符：“我要创建一个目录，结果那里已经有个文件，居然不算错误？”

这种设计导致代码容易隐藏 bug，尤其是在以下场景：

- 路径冲突（文件和目录重名）
- 脚本/工具链中假设“创建失败一定是权限或父目录问题”

P1164R1 的修改内容核心改动：如果路径上已经存在一个非目录的对象，则视为错误。修改后的行为（当前标准）：

1. 成功创建新目录 → 返回 true
2. 路径已存在且是一个目录 → 不报错，返回 false（这是合理行为）
3. 路径已存在但不是目录（是文件、符号链接等）→ 抛出 filesystem_error 异常（或通过 error_code 重载设置错误码）
4. 其他错误（如权限不足、父目录不存在等）→ 正常报错

这个改动使 create_directory() 的语义变得更加直观和安全。相关函数对比

- create_directory(p)：只创建单层目录（父目录必须已存在）。
- create_directories(p)：递归创建多层目录（类似 mkdir -p）。

两者都受到了这个 DR 的影响。示例代码（修改后行为）

cpp

```cpp
#include <filesystem>
#include <iostream>
#include <fstream>

namespace fs = std::filesystem;

int main() {
    fs::path p = "test_path";

    // 创建一个普通文件
    std::ofstream{p} << "hello";

    try {
        bool created = fs::create_directory(p);  // 现在会抛异常！
        std::cout << "Created: " << std::boolalpha << created << '\n';
    } catch (const fs::filesystem_error& e) {
        std::cout << "Error: " << e.what() << '\n';
        std::cout << "Path: " << e.path1() << '\n';
    }
}
```

总结

- DR 名称：P1164R1 — Make create_directory() intuitive
- 问题：原有行为在路径冲突时过于“宽容”，不符合开发者直觉
- 解决方案：把“已存在非目录”提升为错误
- 影响版本：C++17（回溯）+ C++20 正式采纳

这个改动是 C++ 文件系统库走向更实用、更少坑的重要一步。许多早期 C++17 代码在严格模式或新编译器下可能会因为这个 DR 而出现新异常，需要注意错误处理。 

open-std.org

需要对应的 create_directories 行为细节或其他 filesystem DR 的解释吗？