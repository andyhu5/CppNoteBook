# C++23 对 `std::thread::id` 和 `std::stacktrace` 的格式化支持

C++23 为 `std::formatter` 特化增加了对 `std::thread::id` 和 `std::stacktrace`（及 `std::stacktrace_entry`）的支持，使它们可以直接用 `std::format`、`std::print` 等格式化函数输出，而不需要先手动转成字符串。

## 1. `std::thread::id` 的格式化

**C++23 之前**：只能通过 `operator<<` 输出到流：

```cpp
std::thread::id id = std::this_thread::get_id();
std::cout << id << std::endl;  // 只能这样
```

**C++23 之后**：`<thread>` 头文件中新增了 `std::formatter<std::thread::id, CharT>` 特化，可以直接用 `std::format`：

```cpp
#include <thread>
#include <format>
#include <print>  // C++23

std::thread::id id = std::this_thread::get_id();

// 直接格式化
std::string s = std::format("当前线程ID: {}", id);
std::print("当前线程ID: {}\n", id);

// 在多线程日志中很方便
std::print("[线程 {}] 任务开始\n", std::this_thread::get_id());
```

**特点**：

- 该 formatter 输出格式与 `operator<<` 一致（具体格式由实现定义，不保证跨平台一致）
- 不支持自定义格式说明符（format spec 必须为空），例如不能写 `{:>10}` 这种带填充/对齐的格式（部分实现可能有扩展,但标准只保证基本输出）
- 主要意义：统一了 `std::format`/`std::print` 的使用场景，避免混用 `<<` 和 `std::format`

## 2. `std::stacktrace` 的格式化

C++23 引入了 `std::stacktrace`（`<stacktrace>` 头文件），同时提供了对应的 formatter。

```cpp
#include <stacktrace>
#include <format>
#include <print>

void foo() {
    auto trace = std::stacktrace::current();
    
    // 格式化整个调用栈
    std::print("调用栈:\n{}\n", trace);
    
    // 也可以格式化单个栈帧 (stacktrace_entry)
    if (!trace.empty()) {
        std::print("最内层帧: {}\n", trace[0]);
    }
}
```

**输出效果类似**（实现相关，通常包含帧序号、模块、函数、文件:行号）：

```
调用栈:
 0# foo() at main.cpp:10
 1# bar() at main.cpp:20
 2# main at main.cpp:30
```

**关键点**：

- `std::formatter<std::stacktrace_entry, CharT>`：格式化单个栈帧
- `std::formatter<std::stacktrace, CharT>`：格式化整个 trace（内部逐条调用 entry 的格式化，用换行连接）
- 效果等价于调用 `std::to_string(trace)` 或 `operator<<(os, trace)`，但可以嵌入到复杂的格式化字符串中
- 同样：标准未强制规定具体输出的确切格式（因为依赖调试符号、编译器、平台），只是提供了统一接口

## 3. 为什么这个特性有意义

这两个特化本质上都是"胶水"性质的改进，核心价值在于：

1. **统一接口**：C++23 大力推广 `std::format`/`std::print` 作为格式化的统一入口，之前很多标准库类型（如 `thread::id`, `stacktrace`）只支持 `<<`，导致代码风格不统一，现在都能用同一套 `{}` 占位符语法。
2. **组合方便**：可以直接嵌入更复杂的日志格式中：

```cpp
std::print("[{}] [线程 {}] 异常发生:\n{}\n", 
            get_timestamp(), 
            std::this_thread::get_id(), 
            std::stacktrace::current());
```

1. **配合异常处理做调试**：常见用法是配合自定义异常类捕获调用栈：

```cpp
struct traced_exception : std::exception {
    std::stacktrace trace = std::stacktrace::current();
    // ...
};

try {
    throw traced_exception{};
} catch (const traced_exception& e) {
    std::print("异常发生，调用栈:\n{}\n", e.trace);
}
```

## 4. 编译器支持现状（需要注意）

`std::stacktrace` 的支持在各编译器中落地较晚且不完整：

- GCC：需要 GCC 12+，且需要链接 `-lstdc++_libbacktrace`（依赖 libbacktrace）
- MSVC：较早支持较完整
- Clang/libc++：支持相对滞后

由于这块变动较快，如果你需要确认当前（2026年）各编译器/标准库对 `std::stacktrace` formatter 的具体支持情况，我可以帮你搜索最新信息，这样能给你更准确的答案。