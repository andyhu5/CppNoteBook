# std::views::adjacent std::views::pairwise

*std::views::adjacent* is a view similar to *std::views::slide*, producing a sliding window over the input range. 

However, where *std::views::slide* produces **subranges**, 

*std::views::adjacent* produces **tuples** of references to elements.

Consequently, the elements of *std::views::adjacent* can be deconstructed using **structured binding**.

The *std::views::adjacent<2>* has an **alias** in *std::views::pairwise*.

```c++
#include <ranges>
#include <vector>
#include <print>

int main()
{   
    std::vector<int> data{1,2,3,4,5};

    // "sliding tuple" of references to elements
    for (std::tuple<int&,int&,int&> v : data | std::views::adjacent<3>) {
        // iterate over {1,2,3}, {2,3,4}, {3,4,5}
        std::println("({}, {}, {})", std::get<0>(v), std::get<1>(v), std::get<2>(v));
    }

    // deconstructed using structured binding
    for (auto [first, second, third] : data | std::views::adjacent<3>) {
    // iterate over {1,2,3}, {2,3,4}, {3,4,5}
        std::println("({}, {}, {})", first, second, third);
    }

    // std::views::adjacent<2> has an alias
    for (auto [left, right] : data | std::views::pairwise) {
        // iterate over {1,2}, {2,3}, {3,4}, {4,5}
        std::println("({}, {})", left, right);
    }
}

```

