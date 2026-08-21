**C++23 `std::string::contains` / `std::string_view::contains` 详解**

C++23 进一步增强了字符串功能，新增了非常实用的 **`contains()`** 成员函数。

---

### 1. 基本介绍

```cpp
// std::string
bool contains(std::string_view x) const noexcept;
bool contains(CharT x) const noexcept;        // 单个字符

// std::string_view
bool contains(std::string_view x) const noexcept;
bool contains(CharT x) const noexcept;
```

**作用**：判断字符串是否**包含**某个子串或字符。

---

### 2. 使用示例

```cpp
#include <string>
#include <string_view>
#include <iostream>

int main() {
    std::string s = "Hello, C++23 World!";

    std::cout << std::boolalpha;

    // 1. 查找子字符串
    std::cout << s.contains("C++23") << '\n';      // true
    std::cout << s.contains("C++20") << '\n';      // false
    std::cout << s.contains("Hello") << '\n';      // true
    std::cout << s.contains("World!") << '\n';     // true

    // 2. 查找单个字符
    std::cout << s.contains('C') << '\n';          // true
    std::cout << s.contains('!') << '\n';          // true
    std::cout << s.contains('?') << '\n';          // false

    // 3. string_view 也支持
    std::string_view sv = s;
    std::cout << sv.contains("C++23") << '\n';     // true
}
```

---

### 3. 实际应用场景

```cpp
std::string path = "/home/user/document.pdf";

// 文件类型判断
if (path.contains(".pdf")) { ... }

// 日志关键词过滤
if (log.contains("ERROR")) { ... }

// 配置选项检测
if (option.contains("enable")) { ... }

// 用户输入验证
if (input.contains("@") && input.contains(".")) {
    // 可能是邮箱
}
```

---

### 4. 与之前方法的对比

| 写法 (C++23 前)                    | C++23 新写法      | 可读性 | 性能 |
| ---------------------------------- | ----------------- | ------ | ---- |
| `s.find(sub) != std::string::npos` | `s.contains(sub)` | ★★★★★  | 更好 |
| `s.find(ch) != std::string::npos`  | `s.contains(ch)`  | ★★★★★  | 更好 |

**`contains()` 的优势**：
- 代码更清晰、直观
- 语义更明确
- 性能通常更好（库实现可优化）

---

### 5. 注意事项

- **区分大小写**：和 `starts_with` / `ends_with` 一样，不忽略大小写。
- **空字符串**：
  - `s.contains("")` → **始终返回 `true`**
- **性能**：内部通常基于 `find()` 实现，效率很高。
- 支持所有 `basic_string` / `basic_string_view` 的特化（如 `u8string`、`wstring` 等）。

---

### 6. 完整 C++20/23 字符串查询方法对比

| 方法            | C++版本   | 功能              |
| --------------- | --------- | ----------------- |
| `find()`        | C++98     | 查找位置          |
| `starts_with()` | C++20     | 前缀判断          |
| `ends_with()`   | C++20     | 后缀判断          |
| `contains()`    | **C++23** | 子串/字符包含判断 |

---

**推荐写法**：

在新项目中，**强烈推荐**使用 `contains()`，它让字符串判断代码更加现代和易读。

---

需要我展示更多高级用法（如结合 `starts_with`、`ends_with` 做复杂字符串匹配）吗？