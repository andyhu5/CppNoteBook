# C++23 `<spanstream>`：基于 `std::span` 的字符串流

## 背景：为什么需要 spanstream

测试代码：

```c
#include <spanstream>
#include <span>
#include <iostream>

int main() {
    char buffer[64]{};
    
    std::ospanstream out{std::span<char>(buffer)};

    out << "value=" << 42;

    std::cout << buffer << '\n';

    return 0;
}
```

C++ 标准库里已经有两类"内存流"：

1. **`<sstream>`（`std::stringstream` 系列）**：内部持有一个 `std::string`，写入时会**动态分配/拷贝内存**。
2. **`<strstream>`（`std::strstream`，已在 C++98 就被废弃 deprecated，C++26 起被彻底移除）**：直接操作用户提供的 `char*` 缓冲区，避免拷贝，但接口设计古老、不安全（比如所有权语义混乱、缺乏边界安全保证），是一个历史遗留的"坑"。

也就是说，长期以来，如果你想要**流式（`<<`/`>>`）操作一段已有的、固定大小的内存缓冲区**，又不想承担 `stringstream` 的动态分配开销，唯一的选择是使用已废弃、不安全的 `strstream`。

`<spanstream>` 正是为了填补这个空白：提供一个**现代、类型安全、基于 `std::span` 的**替代方案，用来取代 `strstream` 的使用场景。

------

## 核心组件

`<spanstream>` 头文件提供了三个主要类模板（及其 `char`/`wchar_t` 的别名，如通常的 `basic_*` 命名惯例）：

| 类模板                   | 别名（char 版本）  | 用途                        |
| ------------------------ | ------------------ | --------------------------- |
| `std::basic_spanstream`  | `std::spanstream`  | 读写（既能 `<<` 也能 `>>`） |
| `std::basic_ispanstream` | `std::ispanstream` | 只读（`>>`）                |
| `std::basic_ospanstream` | `std::ospanstream` | 只写（`<<`）                |

以及对应的底层 streambuf：`std::basic_spanbuf` / `std::spanbuf`。

## 用法示例

### 写入（ospanstream）

```cpp
#include <spanstream>
#include <iostream>

char buffer[64]{};
std::ospanstream out(std::span<char>(buffer));

out << "value=" << 42;
std::cout << buffer; // "value=42"，直接写入到 buffer，没有额外的堆分配
```

### 读取（ispanstream）

```cpp
char data[] = "123 456";
std::ispanstream in(std::span<char>(data));

int a, b;
in >> a >> b; // a = 123, b = 456
```

### 读写（spanstream）

```cpp
char buf[32]{"hello"};
std::spanstream ss(std::span<char>(buf));

std::string word;
ss >> word;        // 读取 "hello"
ss << " world";     // 从当前位置继续写
```

### 获取已写入内容对应的 span

`spanbuf`/`ospanstream` 提供了 `span()` 成员函数，返回一个反映**当前已写入范围**的 `std::span`（而不是整个底层缓冲区）：

```cpp
std::ospanstream out(std::span<char>(buffer));
out << "abc";
auto written = out.span(); // std::span<char>，大小为 3，指向 "abc"
```

------

## 与 `stringstream` 的关键区别

|          | `stringstream`               | `spanstream`                                                 |
| -------- | ---------------------------- | ------------------------------------------------------------ |
| 底层存储 | 内部 `std::string`，自动扩容 | 用户提供的、**固定大小**的外部缓冲区（`std::span`）          |
| 内存分配 | 内部会 new/realloc           | **不分配任何内存**，纯粹操作已有内存                         |
| 所有权   | 拥有自己的字符串             | **不拥有**内存，只是"借用"（类似 `string_view` 之于 `string`） |
| 越界写入 | 自动扩容，不会溢出           | 缓冲区写满后**停止写入**（不会越界，但也不会自动扩容）       |

也就是说，`spanstream` 更像是 `stringstream` 的"非拥有、零分配"版本，语义上类似 `std::string` 与 `std::string_view` / `std::vector` 与 `std::span` 之间的关系。

------

## 典型应用场景

- **性能敏感场景**：在栈上或预先分配好的缓冲区中做格式化 I/O，避免 `stringstream` 隐藏的堆分配开销（比如高频日志、序列化热路径）。
- **与已有 C 风格缓冲区互操作**：比如网络协议解析中，已经拿到一段 `char[]`/`std::array<char, N>` 缓冲区，想用 `<<`/`>>` 的流式接口去解析/填充，而不需要先拷贝进 `std::string`。
- **替代 `strstream`**：任何还在用已废弃 `std::strstream` 的旧代码，理论上都应该迁移到 `spanstream`，获得类型安全（`std::span` 天然带边界信息）且语义清晰的等价功能。

## 小结

`<spanstream>` 本质上是给"**在固定、非拥有的内存块上做流式格式化 I/O**"这一需求，提供了一个现代、`std::span` 驱动、类型安全的标准方案——用它替代历史遗留、不安全的 `<strstream>`，同时又避免了 `<sstream>` 因动态内存分配带来的性能开销。