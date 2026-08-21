ReadFileEx APC

```c
#include <windows.h>
#include <iostream>

char buffer[128];

VOID CALLBACK ReadDone(
    DWORD error,
    DWORD bytes,
    LPOVERLAPPED
)
{
    if (error == 0)
    {
        buffer[bytes] = '\0';

        std::cout
            << "read:"
            << buffer
            << std::endl;
    }
}


int main()
{
    // 创建并写入文件 开始
    HANDLE hFile = CreateFileA(
        "test.txt",
        GENERIC_WRITE,
        0,
        nullptr,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        nullptr
    );

    const char* text = "hello windows for testing APC";

    DWORD written;

    WriteFile(
        hFile,
        text,
        strlen(text),
        &written,
        nullptr
    );

    std::cout << "written size: " << written << std::endl;

    CloseHandle(hFile);

    // 创建并写入文件 结束



    hFile = CreateFileA(
        "test.txt",
        GENERIC_READ,
        0,
        nullptr,
        OPEN_EXISTING,
        FILE_FLAG_OVERLAPPED,
        nullptr
    );

    OVERLAPPED ov{};

    BOOL ok = ReadFileEx(
        hFile,
        buffer,
        sizeof(buffer) - 1,
        &ov,
        ReadDone
    );


    if (!ok)
    {
        std::cout
            << GetLastError()
            << std::endl;

        return 1;
    }


    std::cout
        << "Read started..."
        << std::endl;


    // 进入 APC 等待
    while (true)
    {
        DWORD r = SleepEx(
            INFINITE,
            TRUE
        );


        if (r == WAIT_IO_COMPLETION)
            break;
    }


    CloseHandle(hFile);
}
```

