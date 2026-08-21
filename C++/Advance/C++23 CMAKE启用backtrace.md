

## 1. 库名问题：GCC 16.1 不再用 `-lstdc++_libbacktrace`

根据 GCC 官方记录：对于 GCC 14 之前的版本，需要链接 -lstdc++_libbacktrace；对于更新的版本，则需要改用 -lstdc++exp。

GCC 16.1 属于"更新版本"，所以正确的库名应该是 **`stdc++exp`**，而不是 `stdc++_libbacktrace`。

## 2. CMake 写法问题

`set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -lxxx")` 这种写法把链接选项塞进了编译标志变量，虽然多数生成器（Makefiles/Ninja）里 `CMAKE_CXX_FLAGS` 也会被传到链接命令行，凑巧能工作，但这不是标准做法，容易导致：

- 纯编译（不链接）阶段也带上这个 `-l`，产生无意义的告警或依赖问题
- 全局污染所有 target，而不是只作用于需要 stacktrace 的目标
- CMake 生成的 `compile_commands.json` 里混入链接参数，干扰 IDE/clangd 解析

正确写法应该用 `target_link_libraries`：

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyApp)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(MyApp main.cpp)

# GCC 14+ 使用 stdc++exp，之前版本用 stdc++_libbacktrace
target_link_libraries(MyApp PRIVATE stdc++exp)

# 建议同时加上 -rdynamic，否则拿不到函数名符号
target_link_options(MyApp PRIVATE -rdynamic)
```

## 3. 前提条件：GCC 需要开启该特性编译

无论库名对不对，还有个前置条件容易被忽略：gcc needs to have been configured with --enable-libstdcxx-backtrace when it was compiled。也就是说，如果你用的是发行版打包的 GCC 16.1（比如 Ubuntu/Debian apt 源），**要看这个发行版编译 GCC 时有没有开这个选项**，否则即便代码写对了、库链接对了，运行时也可能拿不到有效的堆栈信息。

大部分主流发行版（Ubuntu 24.04+、Fedora 较新版本）目前默认是开启的，但如果你是自己从源码编译 GCC，记得加上这个 configure 选项。

## 4. 一个可以直接验证的完整示例

```c++
// main.cpp
#include <stacktrace>
#include <print>

void foo() {
    std::print("调用栈:\n{}\n", std::stacktrace::current());
}

void bar() { foo(); }

int main() {
    bar();
}
```



```cmake
cmake_minimum_required(VERSION 3.20)
project(StacktraceDemo)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(demo main.cpp)
target_link_libraries(demo PRIVATE stdc++exp)
target_link_options(demo PRIVATE -rdynamic)
```



如果链接时报 `cannot find -lstdc++exp`，说明你的 GCC 编译发行版没有构建这个静态库/没开该特性；这时可以退回用 Boost.Stacktrace 或换用支持更完整的发行版工具链。