### std::extents<std::size_t, 2, 3, 4>  —  the shape

```cpp
std::mdspan<int, std::extents<std::size_t, 2, 3, 4>> arr(data);
```

This declaration combines two independent concepts: a shape descriptor (`extents`) and a view over memory (`mdspan`). They are separated deliberately, and understanding why clarifies most confusion around this API.

**Three components of the declaration**

```
std::mdspan< int , std::extents<std::size_t, 2, 3, 4> >( data )
             ^^^    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^   ^^^^
           element        shape descriptor              pointer
            type                                       to memory
```

**1. `int` — element type**

This is the type of each element accessed through the span. Nothing unusual here; it behaves like the `T` in any container.

**2. `std::extents<std::size_t, 2, 3, 4>` — the shape**

`extents` is a standalone type that describes the *shape* of a multidimensional array, independent of any actual memory. It can exist, be named, and be queried without ever touching real data:

```cpp
using Shape3D = std::extents<std::size_t, 2, 3, 4>;

static_assert(Shape3D::rank() == 3);
static_assert(Shape3D::static_extent(0) == 2);
static_assert(Shape3D::static_extent(1) == 3);
static_assert(Shape3D::static_extent(2) == 4);
```

The first template parameter, `std::size_t`, is the index type — the integer type used internally to store and compare indices. The remaining parameters, `2, 3, 4`, are the extents of each dimension, fixed at compile time.

A mental picture of the shape (24 logical elements arranged as 2 x 3 x 4):

```
dim0=0                  dim0=1
+---+---+---+---+      +---+---+---+---+
| 0 | 1 | 2 | 3 |      |12 |13 |14 |15 |   dim1=0
+---+---+---+---+      +---+---+---+---+
| 4 | 5 | 6 | 7 |      |16 |17 |18 |19 |   dim1=1
+---+---+---+---+      +---+---+---+---+
| 8 | 9 |10 |11 |      |20 |21 |22 |23 |   dim1=2
+---+---+---+---+      +---+---+---+---+
   dim2: 0..3              dim2: 0..3
```

The numbers shown are flat offsets into the underlying buffer; `extents` plus the default layout policy (`layout_right`, row-major) is what maps a 3-tuple of indices to one of those offsets.

**3. `data` — the underlying memory**

`mdspan` does not own, allocate, or free memory. It is a non-owning view, exactly like `std::span` but multidimensional. `data` is just a flat buffer of 24 `int`s; `mdspan` overlays the 2x3x4 interpretation on top of it without copying anything:

```cpp
int data[2 * 3 * 4]{};                                  // flat, 24 ints
std::mdspan<int, Shape3D> arr(data);                    // shape overlay, no copy
```

**Why `extents` is a separate type rather than inline integers**

The second template parameter of `mdspan` is constrained to be something satisfying the `extents`-like interface (`rank()`, `extent(i)`, `static_extent(i)`), not a raw pack of integers. There is no library-defined specialization for `mdspan<int, 2, 3, 4>`, which is why that form fails to compile — the compiler looks for a matching partial specialization and finds none.

Separating shape from data buys two things in practice:

```cpp
using Shape3D = std::extents<std::size_t, 2, 3, 4>;

std::mdspan<int,    Shape3D> int_view(int_buffer);      // same shape,
std::mdspan<double, Shape3D> double_view(double_buffer); // different element type
```

and the ability to mix static and dynamic dimensions, using `std::dynamic_extent` as a placeholder for a size known only at runtime:

```cpp
// First dimension determined at runtime, others fixed at compile time.
std::extents<std::size_t, std::dynamic_extent, 3, 4> shape(runtime_rows);
```

This is something an integer-pack template parameter could never express, which is the underlying reason `extents` exists as its own type rather than being folded directly into `mdspan`'s parameter list.

Full demo code:

```c
#include <mdspan>
#include <iostream>
#include <array>      // 可选：使用 std::array 更安全

int main() {
    // 方法1：使用普通 C 数组（最接近你原来的写法）
    int data[2 * 3 * 4]{};   // 初始化为 0

    // 创建 3 维 mdspan： 2 x 3 x 4
    std::mdspan<int, std::extents<std::size_t, 2, 3, 4>> arr(data);

    arr[0, 1, 2] = 999;        // 多维下标访问

    std::cout << "arr[0, 1, 2] = " << arr[0, 1, 2] << '\n';

    // 方法2：使用 std::array（更安全）
    std::array<int, 2 * 3 * 4> data2{};
    std::mdspan<int, std::extents<std::size_t, 2, 3, 4>> arr2(data2.data());

    arr2[1, 2, 3] = 888;
    std::cout << "arr2[1, 2, 3] = " << arr2[1, 2, 3] << '\n';

    return 0;
}
```

