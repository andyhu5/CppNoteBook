# C++23 *std::views::slide*

C++23 *std::views::slide*是包装范围内指定大小的滑动窗口所有位置的视图。

即，大小为 n 的 std::views::slide 的第 i 个元素将是源范围中 [i, i+n) 个元素的子范围。

*请注意，对std::views::slide*生成的滑动窗口进行迭代本质上是*O(m\*n) ，因此请考虑您的用例是否可以在**O(m)*中逐元素实现。

```c++
# include  <ranges>
# include  <vector>

 std::vector< int > data{ 1 , 2 , 3 , 4 , 5 , 6 , 7 , 8 , 9 }; 
	for ( auto window : data | std::views:: slide ( 4 )) { 
    // 遍历所有 4 元素窗口位置：
    // {1,2,3,4} 
    // {2,3,4,5} 
    // {3,4,5,6} 
    // {4,5,6,7} 
    // {5,6,7,8} 
    // {6,7,8,9}
 }
```