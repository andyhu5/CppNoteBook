# C++23：`std::basic_string::resize_and_overwrite`（P1072R10）

## 背景问题：`resize()` 强制的"多余初始化"

`std::string::resize(n)` 在**扩大**字符串时，标准明确要求新增的那部分字符必须先被**值初始化**（`char` 类型的值初始化即填充 `'\0'`），字符串的内容和长度立即变为确定状态：

```cpp
std::string s;
s.resize(64); // 64 个字符全部被初始化为 '\0'
```

这本身是合理的设计——保证任何时候字符串的状态都是良定义的（不存在"未初始化内存暴露给用户"的风险）。但问题在于：**如果你打算立刻用自己的逻辑把这块新内存整个覆盖掉**（比如用 `snprintf`、手写循环填充、调用 C 风格 API 写入数据），那么标准 `resize` 强制做的这次"先清零"操作就变成了**纯粹浪费的开销**——你写入自己的数据之前，CPU 先白白扫了一遍内存去填 `'\0'`。

```cpp
std::string s;
s.resize(64);                              // 第一遍：写入 64 个 '\0'（浪费）
int n = std::snprintf(s.data(), 64, "value=%d", 42); // 第二遍：真正写入的数据
s.resize(n);                                // 再收缩到实际长度
```

在高频调用、大字符串、性能敏感的路径上（比如日志系统、序列化库），这种"多余的一遍清零"是可以被消除的开销。

## C++23 的解决方案：`resize_and_overwrite`

`resize_and_overwrite` 允许你传入一个**"填充器"可调用对象（callable）**，在扩容后：**跳过初始化**，直接把底层缓冲区的指针和容量交给你，由你自己负责填充，并通过返回值告诉字符串"实际写入了多少个字符"。

### 函数签名（概念性描述）

```cpp
template<class Operation>
constexpr void resize_and_overwrite(size_type n, Operation op);
```

- `n`：请求的**最大**长度（也就是可写入的缓冲区大小上限）；
- `op`：一个可调用对象，会被这样调用：`std::move(op)(data(), n)`，其中 `data()` 指向底层字符缓冲区（**内容未初始化，或者说"未指定"，不能假设是 `'\0'`**），`n` 就是你传入的那个长度；
- `op` 的**返回值**必须是可转换为 `size_type` 的类型，代表**实际写入的字符数**，字符串最终的 `size()` 会被设置成这个返回值（相当于自动帮你做了收尾的 `resize`/截断）。

## 使用示例

### 基本用法：配合 `snprintf`

```cpp
#include <string>
#include <cstdio>

std::string s;
s.resize_and_overwrite(64, [](char* buf, std::size_t n) {
    int written = std::snprintf(buf, n, "value=%d", 42);
    return static_cast<std::size_t>(written); // 告诉字符串实际长度是多少
});

// s == "value=42"，且只做了一次写入，没有多余的清零操作
```

### 自定义填充逻辑

```cpp
std::string s;
s.resize_and_overwrite(10, [](char* buf, std::size_t n) {
    std::size_t i = 0;
    for (; i < n && i < 5; ++i) {
        buf[i] = 'A' + i;  // 写入 'A','B','C','D','E'
    }
    return i; // 实际只用了 5 个字符，最终 s.size() == 5
});

// s == "ABCDE"
```

### 对比：`resize` + 手动填充 vs `resize_and_overwrite`

```cpp
// 传统写法：两次开销（清零 + 写入）
std::string s1;
s1.resize(100);
std::size_t len = fill_buffer(s1.data(), 100);
s1.resize(len);

// C++23 写法：只有一次开销（直接写入）
std::string s2;
s2.resize_and_overwrite(100, [](char* buf, std::size_t n) {
    return fill_buffer(buf, n); // fill_buffer 返回实际写入的长度
});
```

## 关键细节与注意事项

### 1. 缓冲区内容"未指定"（unspecified），不能读取未写入的部分

`resize_and_overwrite` 明确**不保证**新分配/新扩展出来的那部分内存是零、是旧数据，还是任何特定值——你**只能写**，不能依赖里面原有的内容做逻辑判断，否则是未定义行为风险（读取"未指定"内容本身不算 UB，但依赖其具体值做逻辑就是错误用法）。

### 2. `op` 不能重新分配、修改字符串的容量或做出其他破坏不变量的操作

`op` 内部**只应该**通过传入的 `buf` 指针和 `n` 写入数据，**不应该**在 `op` 执行期间去调用 `s` 本身的其他成员函数（比如再调用 `s.resize()`、`s.push_back()` 等），因为此时字符串处于"中间的、不完整的状态"，这样做是未定义行为。

### 3. 返回值必须 `<= n`

`op` 返回的实际长度必须不超过你传入的 `n`（缓冲区大小上限），否则行为未定义——这是调用者需要自己保证的契约。

### 4. 对小字符串优化（SSO）同样适用

无论字符串最终是走小字符串优化（存储在栈上/对象内部）还是走堆分配，`resize_and_overwrite` 的语义和优化收益都成立——它避免的是"初始化"这一步，而不是分配本身。

## 典型应用场景

- **高性能日志/序列化库**：格式化输出到字符串缓冲区，追求极致的吞吐量，避免任何不必要的内存写入；
- **封装 C 风格 API**（如各种 `xxx_to_string(char* buf, size_t n)` 风格的函数）到 `std::string` 接口时，避免"先清零、再覆盖"的双重开销；
- **自定义编码/解码逻辑**：需要精确控制写入长度、又想复用 `std::string` 作为存储载体的场景。

这类需求早在标准化之前，很多高性能库（比如 Facebook 的 folly 库中的 `resizeWithoutInitialization`）就已经提供了类似的自定义扩展，C++23 是把这个业界公认有价值的模式**正式纳入标准**。

## 小结

`resize_and_overwrite` 解决的是一个具体而实际的性能痛点：`std::string::resize()` 语义上必须先做"值初始化"，但在**即将被完全覆盖**的场景下，这次初始化是纯粹浪费的开销。C++23 通过让调用者传入一个"填充器"函数，把"分配空间"和"决定实际写入内容/长度"两件事解耦，跳过不必要的初始化步骤，同时依然保证字符串对象最终处于良定义、长度精确的状态——是一项典型的"零开销抽象"式的标准库改进。