这段代码的目的是**验证 `std::decay_t` 的行为是否符合标准规定**。`std::decay` 模拟的是**函数参数按值传递（pass-by-value）\**时编译器对类型所做的转换，因此它不仅去掉 `const`、`volatile` 和引用，还会进行\**数组退化（array-to-pointer decay）\**和\**函数退化（function-to-pointer decay）**。



```c++
template<class T>
struct decay
{
private:
    using U = std::remove_reference_t<T>;

public:
    using type =
        std::conditional_t<std::is_array_v<U>,std::add_pointer_t<std::remove_extent_t<U>>,
        std::conditional_t<std::is_function_v<U>,std::add_pointer_t<U>,std::remove_cv_t<U>>>;
};
```



```c
#include <type_traits>

template<typename T, typename U>
constexpr bool is_decay_equ = std::is_same_v<std::decay_t<T>, U>;

static_assert
(
    is_decay_equ<int, int> &&
    ! is_decay_equ<int, float> &&
    is_decay_equ<int&, int> &&
    is_decay_equ<int&&, int> &&
    is_decay_equ<const int&, int> &&
    is_decay_equ<int[2], int*> &&
    ! is_decay_equ<int[4][2], int*> &&
    ! is_decay_equ<int[4][2], int**> &&
    is_decay_equ<int[4][2], int(*)[2]> &&
    is_decay_equ<int(int), int(*)(int)>
);

int main() {}
```

