**`std::aligned_alloc` 完整解释**（C++17）

`std::aligned_alloc` 是 C++17 在头文件 `<cstdlib>`（或 `<memory>`）中引入的**动态分配对齐内存**的函数。

---

### 1. 函数原型

```cpp
void* std::aligned_alloc(std::size_t alignment, std::size_t size);
```

**参数**：
- `alignment`：请求的对齐字节数（**必须是 2 的幂**，如 8、16、32、64、128 等）
- `size`：要分配的字节数（必须是 `alignment` 的整数倍）

**返回值**：
- 成功时返回**对齐后的内存指针**
- 失败时返回 `nullptr`

**释放**：必须使用 `std::free()` 释放（**不能**用 `delete`）。

---

### 2. 基本示例

```cpp
#include <cstdlib>
#include <memory>
#include <iostream>

int main() {
    // 分配 64 字节对齐的内存，大小为 1024 字节
    void* ptr = std::aligned_alloc(64, 1024);
    
    if (ptr) {
        std::cout << "分配成功，指针 = " << ptr << '\n';
        
        // 使用 std::assume_aligned 提示编译器
        auto aligned_ptr = std::assume_aligned<64>(static_cast<double*>(ptr));
        
        // ... 使用 aligned_ptr 进行计算
        
        std::free(ptr);        // 必须用 free 释放
    }
}
```

---

### 3. 重要规则和限制

1. **`alignment` 必须是 2 的幂**（1, 2, 4, 8, 16, 32, 64...）
2. **`size` 必须是 `alignment` 的整数倍**，否则行为未定义。
3. **不能**和 `new` / `delete` 混用，必须配对使用 `aligned_alloc` + `free`。
4. 不同平台对最大对齐值支持不同（通常至少支持 64 或 128）。
5. C++17 之前常用平台相关函数（如 `_aligned_malloc` on Windows, `posix_memalign` on Linux）。

---

### 4. 与其他对齐工具对比

| 函数 / 特性              | 对齐方式   | 释放方式        | C++版本   | 推荐场景       |
| ------------------------ | ---------- | --------------- | --------- | -------------- |
| `std::malloc`            | 无特殊对齐 | `free`          | C++98     | 普通分配       |
| `std::aligned_alloc`     | 指定对齐   | `free`          | **C++17** | 需要特定对齐时 |
| `new` + `alignas`        | 静态对齐   | `delete`        | C++11     | 对象级别对齐   |
| `std::allocator`         | 可自定义   | 对应 deallocate | C++98     | 容器内部       |
| `std::pmr` (Polymorphic) | 灵活       | -               | C++17     | 高级内存管理   |

---

### 5. 实际应用场景

- **SIMD 编程**（SSE/AVX/AVX-512）
- **高性能计算**、科学模拟
- **缓存行对齐**（避免 false sharing）
- **自定义内存池**
- **GPU / DMA 相关内存分配**
- **音频处理**、**图像处理**等需要对齐的领域

---

### 6. 推荐的现代写法（C++17/20）

```cpp
template <typename T, std::size_t Align = alignof(T)>
T* aligned_new(std::size_t count) {
    std::size_t bytes = count * sizeof(T);
    bytes = (bytes + Align - 1) / Align * Align;  // 向上对齐
    void* p = std::aligned_alloc(Align, bytes);
    return static_cast<T*>(p);
}

// 使用
auto data = aligned_new<double, 64>(1024);
if (data) {
    auto aligned = std::assume_aligned<64>(data);
    // ...
    std::free(data);
}
```

---

**总结**：

- `std::aligned_alloc` 是 C++17 提供的**标准、可移植**的动态对齐内存分配方式。
- 它是 `std::assume_aligned` 发挥作用的重要基础。
- 在性能敏感代码中经常和 `std::assume_aligned` 配合使用。

---

需要我继续讲解 `std::pmr::monotonic_buffer_resource` 等更高级的内存管理技术吗？或者给出 SIMD + aligned_alloc 的完整示例？