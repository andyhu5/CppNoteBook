# C++23：`std::fstream` 的独占模式（Exclusive Mode）

## 背景问题

C++23 之前，`std::fstream`/`std::ofstream` 打开文件时使用的 `std::ios_base::openmode` 标志（`in`, `out`, `app`, `trunc`, `binary`, `ate`）里，没有任何一个能表达"**只有当文件不存在时才创建，如果文件已存在则打开失败**"这个语义——即操作系统层面常见的 `O_EXCL`（POSIX）/ `CREATE_NEW`（Windows）标志对应的能力。

这个能力在很多场景下很重要，典型的比如：

- **防止竞态条件（TOCTOU，Time-Of-Check to Time-Of-Use）**：如果你先用 `std::filesystem::exists()` 检查文件不存在，再用 `trunc` 模式打开创建，这两步之间可能有其他进程/线程创建了同名文件，导致数据被意外覆盖。
- **原子性地创建"锁文件"/唯一文件**：多进程/多线程场景下，需要保证"创建"这个操作本身是原子的。

在 C++23 之前，要实现这个语义，只能绕开标准库、直接调用平台相关的 API（POSIX 的 `open(..., O_CREAT | O_EXCL)` 或 Windows 的 `CreateFileW(..., CREATE_NEW, ...)`），牺牲了可移植性。

## C++23 的解决方案：`std::ios_base::noreplace`

C++23 新增了一个 `openmode` 标志 `std::ios_base::noreplace`，语义是：**要求这次打开操作必须"创建"一个新文件；如果文件已存在，则打开失败**。

```cpp
#include <fstream>
#include <iostream>

int main() {
    std::ofstream file("output.txt", 
                        std::ios::out | std::ios::noreplace);

    if (!file.is_open()) {
        std::cerr << "文件已存在，或创建失败！\n";
        return 1;
    }

    file << "写入新文件成功\n";
    return 0;
}
```

- 如果 `output.txt` **不存在**：文件被创建，打开成功，`file.is_open()` 为 `true`。
- 如果 `output.txt` **已存在**：打开操作**失败**，`file.is_open()` 为 `false`（`failbit` 被设置），文件内容**不会被截断或覆盖**。

## 使用规则/限制

- `noreplace` 必须**与 `out` 一起使用**（否则没有意义，因为只有"输出/创建"语义下才谈得上"是否替换已有文件"）。
- `noreplace` 与 `trunc`（截断已有文件）在语义上是**矛盾**的——`trunc` 意味着"如果文件存在就清空它"，这正是 `noreplace` 想要阻止的行为。同时指定两者是不允许的组合（行为由实现定义，通常应避免这样写）。
- 底层实现上，这对应地会转换为特定平台的原子创建标志：
  - POSIX 系统：`open()` 加上 `O_CREAT | O_EXCL`
  - Windows：`CreateFileW()` 使用 `CREATE_NEW`

## 对比：C++23 之前的"伪安全"写法 vs 之后的原子写法

```cpp
// ❌ C++23 之前：存在竞态条件（TOCTOU）
if (!std::filesystem::exists("output.txt")) {
    std::ofstream file("output.txt"); // 检查和创建之间可能被别的进程插队
    file << "data";
}

// ✅ C++23：原子操作，交给操作系统底层保证
std::ofstream file("output.txt", std::ios::out | std::ios::noreplace);
if (file.is_open()) {
    file << "data";
} else {
    // 文件已存在（或其他打开失败原因），安全地处理这个情况
}
```

## 适用范围

这个特性适用于：

- `std::basic_ofstream`（只写）
- `std::basic_fstream`（读写）
- 以及底层的 `std::basic_filebuf::open()`

对应的构造函数/`open()` 成员函数的 `openmode` 参数都可以传入 `std::ios::noreplace`。

## 小结

`noreplace` 是一个看似很小、但解决了实际工程问题的特性：把"**原子地、排他地创建一个新文件**"这个此前必须依赖平台专属 API 才能安全实现的操作，纳入了标准 `<fstream>` 的能力范围，使得跨平台代码在处理"避免覆盖已有文件"这类需求时，不必再牺牲可移植性或忍受潜在的竞态条件风险。