# std::views::zip {压缩成一个tuple-like的数据}

The C++23 std::views::zip produces a view of **tuple-like** elements, each consisting of the corresponding elements from all the adapted views.

The shortest range determines the number of elements in the view.

The elements of the view maintain reference semantics, meaning that if the arguments are **mutable** ranges, their elements can be mutated through the elements of this view.



```c++
#include <vector>
#include <ranges>
#include <iomanip>
#include <iostream>

int main() {
    std::vector<int> first{1,2,3,4,5};
    std::vector<double> second{9,8,7,6};

    // Iterate over the elements of the zip view
    for (auto [a, b] : std::views::zip(first, second)) {
        // {1,9}, {2,8}, {3,7}, {4,6}
        std::cout << a << " " << b << "\n";
    }
    std::cout << "\n";

    // Same as above, without structured binding
    for (std::tuple<int&,double&> el : std::views::zip(first, second)) {
        // {1,9}, {2,8}, {3,7}, {4,6}
        std::cout << std::get<0>(el) << " " << std::get<1>(el) << "\n";
    }
    std::cout << "\n";
    
    std::vector<std::string> third{"label1", "label2", "label3"};

    // The zip view can accept one or more arguments
    for (auto [a, b, c] : std::views::zip(first, second, third)) {
        // {1,9,"label1"}, {2,8,"label2"}, {3,7,"label3"}
        std::cout << a << " " << b << " " << std::quoted(c) << "\n";
    }
    std::cout << "\n";

    // We can also modify the original ranges through the tuple
    for (auto [a, b, c] : std::views::zip(first, second, third)) {
        a = a + b;
        std::cout << a << " " << b << " " << std::quoted(c) << "\n";
    }
    std::cout << "\n";
    // first == {10, 10, 10, 4, 5}

    for (auto v : first)
        std::cout << v << " ";
    std::cout << "\n";
}
```

