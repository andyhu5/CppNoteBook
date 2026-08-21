

WaitOnAddress

WakeByAddressAll

WakeByAddressSingle

```c
#include <windows.h>
#include <iostream>
#include <thread>
int data = 0;
void consumer()
{
    int old = 0;
    std::cout << "waiting...\n";
    WaitOnAddress(
        &data,
        &old,
        sizeof(data),
        INFINITE
    );
    std::cout << "data = "
        << data
        << "\n";
}

void producer()
{
    Sleep(5000);
    data = 42;
    //WakeByAddressSingle(&data);
    WakeByAddressAll(&data);
}

int main()
{
    std::jthread t1(consumer);
    std::jthread t2(consumer);
    std::jthread t3(producer);
}
```

