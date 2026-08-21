`std::ranges::fold_right` 和 `fold_left` 的二元操作函数**参数顺序是相反的**——这是最容易踩坑的地方。

## ***注意***: lambda 参数顺序

- `fold_left` 的调用方式是：`f(累加值, 元素)`，即 `f(acc, x)`
- `fold_right` 的调用方式是：`f(元素, 累加值)`，即 `f(x, acc)`

具体展开：

```
fold_right(x1, x2, ..., xn, init) 
= f(x1, f(x2, ..., f(xn, init)))
```

也就是从**最后一个元素**开始，先跟 `init` 结合，再一步步往前结合。

你的 lambda 写成了 `[](std::string acc, int x)`，参数顺序对应的是 `fold_left` 的约定，而不是 `fold_right` 的约定，所以类型不匹配（实际调用时是 `f(int, string)`），无法通过编译。

## 修复代码

```cpp
#include <numeric>
#include <ranges>
#include <vector>
#include <string>
#include <iostream>

int main() {
    std::vector<int> v = { 1, 2, 3, 4, 5 };

    // 求和：std::plus<> 两个参数都是 int，交换顺序结果不变，本身没问题
    auto sum = std::ranges::fold_right(v, 0, std::plus<>());
    std::cout << sum << '\n'; // 15

    // 拼接成字符串：参数顺序要改成 (元素, 累加值)
    auto s = std::ranges::fold_right(v, std::string{},
        [](int x, std::string acc) {
            return std::to_string(x) + "," + acc;  // 注意：元素在前拼接
        });
    std::cout << s << '\n'; // "1,2,3,4,5,"
}
```

## 为什么这样拼接是对的

`fold_right` 是从右往左处理，所以过程是：

```
acc = ""                      (init)
处理 5: acc = "5" + "," + ""       = "5,"
处理 4: acc = "4" + "," + "5,"     = "4,5,"
处理 3: acc = "3" + "," + "4,5,"   = "3,4,5,"
处理 2: acc = "2" + "," + "3,4,5," = "2,3,4,5,"
处理 1: acc = "1" + "," + "2,3,4,5," = "1,2,3,4,5,"
```

最终结果就是 `"1,2,3,4,5,"`，和期望输出一致。

## 小结对比

| 算法         | lambda 参数顺序 | 处理方向 |
| ------------ | --------------- | -------- |
| `fold_left`  | `f(acc, x)`     | 从左到右 |
| `fold_right` | `f(x, acc)`     | 从右到左 |

如果你的二元操作是**可交换**的（比如 `std::plus<>`），参数顺序不影响结果，所以求和那部分本来就能正常工作；但如果操作是**不可交换**的（比如字符串拼接、`std::minus<>` 等），参数顺序写反就会导致编译错误或结果错误，这也是 `fold_right` 相比 `fold_left` 更容易写错的地方。