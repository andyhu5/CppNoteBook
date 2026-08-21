Windows 下 `std::string` 和 `std::wstring` 的转换，本质上是**字符编码转换**。

通常推荐使用 Windows API `MultiByteToWideChar()` 和 `WideCharToMultiByte()`，而不是简单的强制类型转换。

如果你的 `std::string` 是 UTF-8，这是最常用的写法。

```cpp
#include <windows.h>
#include <string>

std::wstring ToWide(const std::string& str)
{
    if (str.empty())
        return {};

    int size = MultiByteToWideChar(
        CP_UTF8,
        0,
        str.data(),
        static_cast<int>(str.size()),
        nullptr,
        0);

    if (size <= 0)
        return {};

    std::wstring result(size, L'\0');

    MultiByteToWideChar(
        CP_UTF8,
        0,
        str.data(),
        static_cast<int>(str.size()),
        result.data(),
        size);

    return result;
}
```

反过来：

```cpp
std::string ToUtf8(const std::wstring& str)
{
    if (str.empty())
        return {};

    int size = WideCharToMultiByte(
        CP_UTF8,
        0,
        str.data(),
        static_cast<int>(str.size()),
        nullptr,
        0,
        nullptr,
        nullptr);

    if (size <= 0)
        return {};

    std::string result(size, '\0');

    WideCharToMultiByte(
        CP_UTF8,
        0,
        str.data(),
        static_cast<int>(str.size()),
        result.data(),
        size,
        nullptr,
        nullptr);

    return result;
}
```

例如：

```cpp
std::string text = "你好，Windows";

std::wstring wtext = ToWide(text);

std::string text2 = ToUtf8(wtext);
```

最终：

```text
std::string
    │
    │ UTF-8
    ▼
std::wstring
    │
    │ UTF-16
    ▼
std::string
```

在 Windows API 中：

```cpp
WinHttpConnect(
    m_session,
    host.c_str(),
    ...
);
```

这里使用的是 `LPCWSTR`，所以你的 `host` 使用：

```cpp
std::wstring
```

是正确的。

而 `WinHttpReadData()` 得到的是：

```cpp
BYTE*
```

本质上是**原始字节流**，所以用：

```cpp
std::string
```

保存 HTTP response body 也比较合适。

例如你的设计：

```cpp
std::string WinHttpClient::Get(
    const std::wstring& host,
    const std::wstring& path,
    unsigned short port);
```

是比较合理的：

```text
host/path
    ↓
std::wstring
    ↓
WinHTTP Windows API
    ↓
HTTP response bytes
    ↓
std::string
```

如果 response 是 UTF-8 JSON，例如：

```json
{"message":"你好"}
```

那么 `std::string` 直接保存 UTF-8 字节即可，**不需要先转换成 `std::wstring`**。

只有当你需要使用 Windows 的 Unicode API，或者需要以 `std::wstring` 处理中文文本时，才进行：

```cpp
std::wstring text = ToWide(response);
```

另外，Windows 下有一个容易混淆的点：`std::wstring` 在 Windows 通常是 **UTF-16**，而不是 UTF-8；`std::string` 本身也**没有规定编码**，它只是字节字符串。实际工程中最好明确约定：`std::string = UTF-8`，`std::wstring = Windows UTF-16`。