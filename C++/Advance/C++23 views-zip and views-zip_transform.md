

### C++23 views:zip and views::zip_transform

```c#
#include <ranges>
#include <print>
#include <vector>

int main() {
    std::vector<int> v1 = {1, 2, 3, 4, 5};

    std::vector<int> v2 {7,8,9,10,11};

    //std::ranges::views::zip
    auto rr11 = std::ranges::zip_view(v1, v2) | std::views::common;
    for (const auto& [a, b] : rr11) {
        std::println("({}, {})", a, b);
    }

    //std::ranges::views::zip_transform
    auto rr2 = std::ranges::zip_transform_view([](int a, int b) { return a + b; }, v1, v2);
    for (const auto& x : rr2) {
        std::println("{}", x);
    }
}
```

