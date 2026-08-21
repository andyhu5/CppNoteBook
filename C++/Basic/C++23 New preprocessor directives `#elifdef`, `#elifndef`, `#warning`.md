## C++23 New preprocessor directives: `#elifdef`, `#elifndef`, `#warning`

C++23 新增了三个预处理指令，来自提案 **P2334R1**，目的是让条件编译更加简洁、可读。

| 指令              | 等价写法                |
| ----------------- | ----------------------- |
| `#elifdef MACRO`  | `#elif defined(MACRO)`  |
| `#elifndef MACRO` | `#elif !defined(MACRO)` |
| `#warning`        | 输出编译警告信息        |

### 1. `#elifdef`

以前：

```cpp
#ifdef _WIN32
    // Windows
#elif defined(__linux__)
    // Linux
#elif defined(__APPLE__)
    // macOS
#endif
```

C++23：

```cpp
#ifdef _WIN32
    // Windows
#elifdef __linux__
    // Linux
#elifdef __APPLE__
    // macOS
#endif
```

功能完全相同，只是更简洁。

------

### 2. `#elifndef`

以前：

```cpp
#ifdef DEBUG
    debug();
#elif !defined(TEST)
    release();
#endif
```

C++23：

```cpp
#ifdef DEBUG
    debug();
#elifndef TEST
    release();
#endif
```

等价于：

```cpp
#elif !defined(TEST)
```

------

### 3. `#warning`

用于在编译期间输出一条**警告**，但**不会终止编译**。

```cpp
#warning "This API is deprecated."
```

编译器可能输出：

```text
warning: This API is deprecated.
```

适合提醒：

- 使用了废弃 API
- 开启了实验功能
- 某个平台尚未完全支持
- 某些配置可能影响性能

例如：

```cpp
#ifndef NDEBUG
#warning "Debug build."
#endif
```

------

### 与 `#error` 的区别

| 指令       | 是否终止编译 |
| ---------- | ------------ |
| `#warning` | ❌ 否         |
| `#error`   | ✅ 是         |

例如：

```cpp
#ifndef __cpp_consteval
#error "C++20 consteval required."
#endif
```

这里会直接停止编译。

------

### 综合示例

```cpp
#ifdef _WIN32
    constexpr auto os = "Windows";
#elifdef __linux__
    constexpr auto os = "Linux";
#elifdef __APPLE__
    constexpr auto os = "macOS";
#else
#warning "Unknown platform."
    constexpr auto os = "Unknown";
#endif
```

------

### 编译器支持

| 编译器 | 支持版本                    |
| ------ | --------------------------- |
| GCC    | 13+                         |
| Clang  | 16+                         |
| MSVC   | VS 2022 17.7+（C++23 模式） |

------

### 总结

- `#elifdef`：`#elif defined(MACRO)` 的简写。
- `#elifndef`：`#elif !defined(MACRO)` 的简写。
- `#warning`：输出编译警告，不终止编译，类似于 `#pragma message`，但属于标准 C++ 预处理指令。

这三个特性都属于**语法改进**，不会影响运行时性能，但能让跨平台和条件编译代码更加清晰、统一。