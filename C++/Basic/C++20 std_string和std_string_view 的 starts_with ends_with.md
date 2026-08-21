**C++20 `std::string` 和 `std::string_view` 的 `starts_with` / `ends_with` 详解**

C++20 为字符串处理新增了两个非常方便的成员函数，大幅提升了代码可读性。

---

### 1. 成员函数概览

适用于以下类型（C++20）：

- `std::string`
- `std::string_view`
- `std::wstring` / `std::u8string` / `std::u16string` / `std::u32string` 等所有 `std::basic_string` 和 `std::basic_string_view`

#### 函数签名

```cpp
// starts_with
bool starts_with(const std::string& prefix) const noexcept;
bool starts_with(std::string_view prefix) const noexcept;
bool starts_with(CharT ch) const noexcept;           // 单个字符

// ends_with
bool ends_with(const std::string& suffix) const noexcept;
bool ends_with(std::string_view suffix) const noexcept;
bool ends_with(CharT ch) const noexcept;             // 单个字符
```

---

### 2. 使用示例

```cpp
#include <string>
#include <string_view>
#include <iostream>

int main() {
    std::string s = "Hello, C++20 World!";
    std::string_view sv = s;

    // starts_with
    std::cout << std::boolalpha;
    std::cout << s.starts_with("Hello") << '\n';        // true
    std::cout << s.starts_with("hello") << '\n';        // false（区分大小写）
    std::cout << s.starts_with('H') << '\n';            // true
    std::cout << sv.starts_with("C++") << '\n';         // false

    // ends_with
    std::cout << s.ends_with("World!") << '\n';         // true
    std::cout << s.ends_with("world!") << '\n';         // false
    std::cout << s.ends_with('!') << '\n';              // true

    // string_view 同样支持
    std::cout << sv.ends_with("C++20 World!") << '\n';  // true
}
```

---

### 3. 实际应用场景

```cpp
// 1. 文件后缀判断
std::string filename = "report.pdf";
if (filename.ends_with(".pdf")) {
    // 处理 PDF
}

// 2. URL / 协议判断
std::string url = "https://example.com";
if (url.starts_with("https://")) { ... }

// 3. 命令行参数前缀
if (arg.starts_with("--verbose")) { ... }

// 4. 字符串分类
if (line.starts_with('#')) { /* 注释 */ }
```

---

### 4. 优点对比（与旧代码对比）

| 写法（C++20 前）                           | C++20 新写法            | 可读性 |
| ------------------------------------------ | ----------------------- | ------ |
| `s.substr(0, prefix.size()) == prefix`     | `s.starts_with(prefix)` | ★★★★★  |
| `s.compare(0, suffix.size(), suffix) == 0` | `s.ends_with(suffix)`   | ★★★★★  |
| `s.back() == ch`                           | `s.ends_with(ch)`       | ★★★★   |

---

### 5. 注意事项

- **区分大小写**：不进行任何大小写转换。
- **空字符串**：
  - `s.starts_with("")` → 永远返回 `true`
  - `s.ends_with("")` → 永远返回 `true`
- **性能**：非常高效，通常是 O(M)（M 为前/后缀长度），内部实现高度优化。
- **支持 `std::string_view`**：可以直接传入 `string_view`，避免不必要的拷贝。

---

### 6. 扩展：`contains`（C++23）

C++23 进一步增加了：

```cpp
bool contains(std::string_view x) const noexcept;
bool contains(CharT x) const noexcept;
```

---

**总结**：

`starts_with()` 和 `ends_with()` 是 C++20 中**最实用、最受欢迎**的字符串增强之一，让代码更加清晰、现代。

推荐在新代码中**大量使用**，替代以前繁琐的 `substr` + `compare` 写法。

---

需要我给出更多实际项目中的用法（如文件处理、协议解析、配置解析等）吗？