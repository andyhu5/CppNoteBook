IOCP

```
#include <windows.h>
#include <iostream>
#include <cstring>

char buffer[128];

int main()
{
    // ============================================================
    // 1. 创建并写入文件
    // ============================================================

    HANDLE hFile = CreateFileA(
        "test.txt",
        GENERIC_WRITE,
        0,
        nullptr,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        nullptr
    );

    if (hFile == INVALID_HANDLE_VALUE)
    {
        std::cout << "CreateFile(write) failed: "
            << GetLastError()
            << std::endl;

        return 1;
    }

    const char* text = "hello windows for testing IOCP";

    DWORD written = 0;

    if (!WriteFile(
        hFile,
        text,
        static_cast<DWORD>(strlen(text)),
        &written,
        nullptr))
    {
        std::cout << "WriteFile failed: "
            << GetLastError()
            << std::endl;

        CloseHandle(hFile);
        return 1;
    }

    std::cout
        << "written size: "
        << written
        << std::endl;

    CloseHandle(hFile);


    // ============================================================
    // 2. 重新打开文件
    // ============================================================

    hFile = CreateFileA(
        "test.txt",
        GENERIC_READ,
        0,
        nullptr,
        OPEN_EXISTING,
        FILE_FLAG_OVERLAPPED,
        nullptr
    );

    if (hFile == INVALID_HANDLE_VALUE)
    {
        std::cout << "CreateFile(read) failed: "
            << GetLastError()
            << std::endl;

        return 1;
    }


    // ============================================================
    // 3. 创建 IOCP
    // ============================================================

    HANDLE hIOCP = CreateIoCompletionPort(
        INVALID_HANDLE_VALUE,
        nullptr,
        0,
        0
    );

    if (hIOCP == nullptr)
    {
        std::cout << "CreateIoCompletionPort failed: "
            << GetLastError()
            << std::endl;

        CloseHandle(hFile);
        return 1;
    }


    // ============================================================
    // 4. 将文件句柄绑定到 IOCP
    // ============================================================
    const char* key = "hello world";

    HANDLE result = CreateIoCompletionPort(
        hFile,
        hIOCP,
        reinterpret_cast<ULONG_PTR>(key),
        0
    );

    if (result == nullptr)
    {
        std::cout << "Associate IOCP failed: "
            << GetLastError()
            << std::endl;

        CloseHandle(hIOCP);
        CloseHandle(hFile);
        return 1;
    }


    // ============================================================
    // 5. 准备 OVERLAPPED
    // ============================================================

    OVERLAPPED ov{};

    // 从文件开头开始读取
    ov.Offset = 0;
    ov.OffsetHigh = 0;


    // ============================================================
    // 6. 发起异步读取
    // ============================================================

    BOOL ok = ReadFile(
        hFile,
        buffer,
        sizeof(buffer) - 1,
        nullptr,
        &ov
    );


    if (!ok)
    {
        DWORD error = GetLastError();

        if (error != ERROR_IO_PENDING)
        {
            std::cout
                << "ReadFile failed: "
                << error
                << std::endl;

            CloseHandle(hIOCP);
            CloseHandle(hFile);
            return 1;
        }

        std::cout
            << "ReadFile pending..."
            << std::endl;
    }
    else
    {
        std::cout
            << "ReadFile completed immediately"
            << std::endl;
    }


    // ============================================================
    // 7. 等待 IOCP 完成通知
    // ============================================================

    DWORD bytesTransferred = 0;

    ULONG_PTR completionKey = 0;

    LPOVERLAPPED completedOverlapped = nullptr;


    BOOL ret = GetQueuedCompletionStatus(
        hIOCP,
        &bytesTransferred,
        &completionKey,
        &completedOverlapped,
        INFINITE
    );


    if (!ret)
    {
        std::cout
            << "GetQueuedCompletionStatus failed: "
            << GetLastError()
            << std::endl;

        CloseHandle(hIOCP);
        CloseHandle(hFile);
        return 1;
    }


    // ============================================================
    // 8. IO 完成
    // ============================================================

    buffer[bytesTransferred] = '\0';

    std::cout
        << "IO completed"
        << std::endl;

    std::cout
        << "CompletionKey = "
        <<reinterpret_cast<const char*>(completionKey)
        << std::endl;

    std::cout
        << "Read bytes = "
        << bytesTransferred
        << std::endl;

    std::cout
        << "Read data = "
        << buffer
        << std::endl;


    // ============================================================
    // 9. 清理
    // ============================================================

    CloseHandle(hIOCP);

    CloseHandle(hFile);

    return 0;
}
```

