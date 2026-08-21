SleepEx 演示APC

```c
#include <windows.h>
#include <iostream>

// APC 回调函数
VOID CALLBACK MyAPC(
    ULONG_PTR param
)
{
    auto message =
        reinterpret_cast<const char*>(param);
    std::cout << "APC callback executed, param="
        << message
        << std::endl;
}

// 工作线程
DWORD WINAPI Worker(
    LPVOID
)
{
    std::cout << "Worker thread sleep..."
        << std::endl;

    DWORD ret = SleepEx(
        INFINITE,
        TRUE        // 允许执行 APC
    );


    if (ret == WAIT_IO_COMPLETION)
    {
        std::cout << "SleepEx interrupted by APC"
            << std::endl;
    }

    return 0;
}


int main()
{
    DWORD tid;

    HANDLE hThread = CreateThread(
        nullptr,
        0,
        Worker,
        nullptr,
        0,
        &tid
    );


    // 等待工作线程进入 SleepEx
    Sleep(1000);


    std::cout << "Queue APC..."
        << std::endl;


    const char* message = "hello world";


    QueueUserAPC(
        MyAPC,
        hThread,
        reinterpret_cast<ULONG_PTR>(message)
    );


    WaitForSingleObject(
        hThread,
        INFINITE
    );


    CloseHandle(hThread);

    return 0;
}
```

