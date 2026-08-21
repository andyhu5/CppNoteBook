**C++20 `std::c8rtomb` 和 `std::mbrtoc8` 完整解释**

这两个函数是 C++20 为 **`char8_t`**（UTF-8）专门新增的**多字节字符转换函数**，定义在头文件 **`<cuchar>`** 中。

它们是 `mbrtowc` / `wcrtomb` 的 UTF-8 版本，用于在**本地多字节编码**（narrow multibyte encoding）和 **UTF-8** 之间进行转换。

---

### 1. `std::mbrtoc8` —— 多字节 → UTF-8

```cpp
std::size_t mbrtoc8(char8_t* pc8, 
                    const char* s, 
                    std::size_t n, 
                    std::mbstate_t* ps);
```

**作用**：将**本地多字节字符**（由当前 locale 决定）转换为 **UTF-8** 编码（`char8_t`）。

#### 参数说明
- `pc8`：输出 UTF-8 码元（code unit）的指针
- `s`：输入的多字节字符序列
- `n`：最多检查 `s` 中的字节数
- `ps`：转换状态（必须使用同一个 `mbstate_t` 处理多码元字符）

#### 返回值
- `0`：转换成功且是空字符 (`'\0'`)
- `> 0`：成功转换的输入字节数
- `-3`：输出了一个 UTF-8 序列中的后续码元（需继续调用）
- `-2`：输入不完整（需要更多字节）
- `-1`：编码错误（设置 `errno = EILSEQ`）

---

### 2. `std::c8rtomb` —— UTF-8 → 多字节

```cpp
std::size_t c8rtomb(char* s, 
                    char8_t c8, 
                    std::mbstate_t* ps);
```

**作用**：将单个 **UTF-8 码元** 转换为**本地多字节字符**。

#### 参数说明
- `s`：输出多字节字符的缓冲区（至少 `MB_CUR_MAX` 大小）
- `c8`：输入的 UTF-8 码元
- `ps`：转换状态

#### 返回值
- 成功时：写入的字节数（包括 shift sequences）
- 如果 `c8` 不是一个完整 UTF-8 字符的最后一个码元，则不写入，只更新状态
- 错误时：返回 `(size_t)-1`，设置 `errno = EILSEQ`

---

### 3. 使用示例

```cpp
#include <cuchar>
#include <iostream>
#include <string>
#include <locale>

int main() {
    std::setlocale(LC_ALL, "");   // 设置本地 locale

    std::mbstate_t state{};
    char8_t u8char;
    const char* mb = "中";        // 本地多字节字符

    // 多字节 → UTF-8
    std::size_t ret = std::mbrtoc8(&u8char, mb, 10, &state);
    if (ret > 0) {
        std::cout << "成功转换为 UTF-8\n";
    }

    // UTF-8 → 多字节
    char buffer[MB_CUR_MAX];
    state = {};  // 重置状态
    std::size_t ret2 = std::c8rtomb(buffer, u8char, &state);
    if (ret2 != (std::size_t)-1) {
        buffer[ret2] = '\0';
        std::cout << "转换回多字节: " << buffer << '\n';
    }
}
```

---

### 4. 为什么需要这两个函数？

- C++20 引入 `char8_t` 后，需要**标准、可移植**的方式在 UTF-8 和本地编码之间转换。
- 之前人们常用 `char` 滥用表示 UTF-8，现在有了**类型安全**的转换函数。
- 处理**多字节 locale**（如中文、日文环境）时非常重要。

---

### 5. 注意事项

- 必须**正确管理 `std::mbstate_t`** 状态，尤其处理多字节/多码元字符时。
- 依赖**当前 locale**（使用 `std::setlocale` 或 `std::locale`）。
- 错误处理很重要（`-1` 返回值 + `errno`）。
- 推荐配合 `std::u8string` / `std::u8string_view` 使用。
- 在现代代码中，如果只需要 UTF-8 处理，优先考虑 `std::codecvt` 或第三方库（如 ICU），但这两个函数是标准底层机制。

---

**总结**：

- `mbrtoc8`：**多字节 → UTF-8** (`char` → `char8_t`)
- `c8rtomb`：**UTF-8 → 多字节** (`char8_t` → `char`)

这是 C++20 对 UTF-8 原生支持的重要组成部分。

需要我给出更完整的转换 `std::string` ↔ `std::u8string` 的实用包装函数示例吗？