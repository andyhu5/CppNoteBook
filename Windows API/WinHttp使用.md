可以。`WinHttpReceiveResponse()` 只是表示“HTTP Response 的 headers 已经收到”，**不会自动把 response body 读出来**。

需要继续调用：

```cpp
WinHttpQueryDataAvailable()
WinHttpReadData()
```

典型流程是：

```text
WinHttpSendRequest()
        ↓
WinHttpReceiveResponse()
        ↓
WinHttpQueryDataAvailable()
        ↓
WinHttpReadData()
        ↓
打印 response body
```

你的代码可以直接修改成：

```cpp
bool WinHttpClient::Get(
    const std::wstring& host,
    const std::wstring& path,
    unsigned short port)
{
    HINTERNET hConnect =
        WinHttpConnect(
            m_session,
            host.c_str(),
            port,
            0);

    if (!hConnect)
        return false;

    HINTERNET hRequest =
        WinHttpOpenRequest(
            hConnect,
            L"GET",
            path.c_str(),
            nullptr,
            WINHTTP_NO_REFERER,
            WINHTTP_DEFAULT_ACCEPT_TYPES,
            0);

    if (!hRequest)
    {
        WinHttpCloseHandle(hConnect);
        return false;
    }

    BOOL result =
        WinHttpSendRequest(
            hRequest,
            WINHTTP_NO_ADDITIONAL_HEADERS,
            0,
            WINHTTP_NO_REQUEST_DATA,
            0,
            0,
            0);

    if (result)
    {
        result =
            WinHttpReceiveResponse(
                hRequest,
                nullptr);
    }

    if (!result)
    {
        WinHttpCloseHandle(hRequest);
        WinHttpCloseHandle(hConnect);
        return false;
    }

    // 读取 response body
    DWORD size = 0;

    do
    {
        size = 0;

        if (!WinHttpQueryDataAvailable(
                hRequest,
                &size))
        {
            WinHttpCloseHandle(hRequest);
            WinHttpCloseHandle(hConnect);
            return false;
        }

        if (size == 0)
            break;

        std::vector<char> buffer(size + 1);

        DWORD bytesRead = 0;

        if (!WinHttpReadData(
                hRequest,
                buffer.data(),
                size,
                &bytesRead))
        {
            WinHttpCloseHandle(hRequest);
            WinHttpCloseHandle(hConnect);
            return false;
        }

        buffer[bytesRead] = '\0';

        std::println("{}", buffer.data());

    } while (size > 0);

    WinHttpCloseHandle(hRequest);
    WinHttpCloseHandle(hConnect);

    return true;
}
```

需要增加：

```cpp
#include <windows.h>
#include <winhttp.h>
#include <vector>
#include <print>
```

并链接：

```text
winhttp.lib
```

例如服务器返回：

```json
{
    "name": "Andy",
    "version": "1.0"
}
```

那么：

```cpp
WinHttpReadData(...)
```

每次可能得到其中的一部分：

```text
第一次：
{
    "name": "Andy",

第二次：
    "version": "1.0"
}
```

所以这里**不能假设一次 `WinHttpReadData()` 就得到完整 response**，必须循环读取。

另外，如果你想打印 HTTP 状态码，例如 `200`、`404`、`500`，可以在 `WinHttpReceiveResponse()` 后使用：

```cpp
DWORD statusCode = 0;
DWORD statusCodeSize = sizeof(statusCode);

WinHttpQueryHeaders(
    hRequest,
    WINHTTP_QUERY_STATUS_CODE |
    WINHTTP_QUERY_FLAG_NUMBER,
    WINHTTP_HEADER_NAME_BY_INDEX,
    &statusCode,
    &statusCodeSize,
    WINHTTP_NO_HEADER_INDEX);

std::println("HTTP Status: {}", statusCode);
```

于是完整的处理逻辑就是：

```text
WinHttpSendRequest()
        │
        ▼
WinHttpReceiveResponse()
        │
        ├── WinHttpQueryHeaders()
        │       └── HTTP 200
        │
        ▼
WinHttpQueryDataAvailable()
        │
        ▼
WinHttpReadData()
        │
        ├── chunk 1
        ├── chunk 2
        ├── chunk 3
        └── ...
        │
        ▼
      response body
```

如果你是在写一个 `WinHttpClient` 类，我更建议把 `WinHttpReadData()` 的结果最终累积成一个 `std::string` 再返回，而不是在 `Get()` 内部直接 `println`。这样 `Get()` 才是真正的 HTTP client API，调用者可以自己决定是打印、解析 JSON，还是保存到文件。