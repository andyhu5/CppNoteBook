**C++20 `std::pmr::polymorphic_allocator` 增强详解**

C++20 对 `std::pmr::polymorphic_allocator` 进行了重要改进，使其成为更强大、通用的内存分配工具。

### 1. 默认模板参数改为 `std::byte`

**C++17**：
```cpp
template <class T> class polymorphic_allocator;
```

**C++20**：
```cpp
template <class T = std::byte> class polymorphic_allocator;
```

**意义**：
- `std::pmr::polymorphic_allocator<>`（空模板参数）现在等价于 `polymorphic_allocator<std::byte>`。
- 这使得它成为**通用的字节分配器**（vocabulary type），非常适合分配原始内存、缓冲区，而不需要指定具体类型。
- 极大提升了代码的通用性，尤其在 PMR（Polymorphic Memory Resource）场景中。

```cpp
std::pmr::polymorphic_allocator<> alloc;  // C++20 推荐写法，等价于 std::byte
```

---

### 2. C++20 新增成员函数（最重要增强）

C++20 为 `polymorphic_allocator` 添加了一系列**低级内存管理**函数，让它不仅能分配对象，还能方便地分配**原始字节内存**。

#### (1) `allocate_bytes` / `deallocate_bytes`

```cpp
void* allocate_bytes(std::size_t nbytes, 
                     std::size_t alignment = alignof(std::max_align_t));

void deallocate_bytes(void* p, 
                      std::size_t nbytes, 
                      std::size_t alignment = alignof(std::max_align_t));
```

- 用于分配/释放**原始对齐内存**（类似 `std::aligned_alloc` 但通过 memory resource）。
- 常用于实现自定义容器、缓冲区、内存池等。

#### (2) `new_object` / `delete_object`（强烈推荐）

```cpp
template <class U, class... CtorArgs>
U* new_object(CtorArgs&&... args);

template <class U>
void delete_object(U* p);
```

- `new_object`：分配内存 + 使用 allocator 构造对象（自动处理 allocator-aware 类型）。
- `delete_object`：析构对象 + 释放内存。

**示例**：

```cpp
std::pmr::monotonic_buffer_resource mr{1024 * 1024};
std::pmr::polymorphic_allocator<> alloc{&mr};

auto* p = alloc.new_object<std::pmr::string>("Hello C++20 PMR");
std::cout << *p << '\n';

alloc.delete_object(p);   // 自动析构 + 释放
```

#### (3) 其他新增/改进函数

- `allocate_object` / `deallocate_object`
- `construct` / `destroy` 的行为改进
- 更好的 `make_obj_using_allocator` 配合支持

---

### 3. 完整示例

```cpp
#include <memory_resource>
#include <vector>
#include <string>
#include <iostream>

int main() {
    // 1. 使用 monotonic_buffer_resource（栈上快速分配）
    std::pmr::monotonic_buffer_resource pool{4096};
    std::pmr::polymorphic_allocator<> alloc{&pool};

    // 2. 使用通用 allocator<> (std::byte)
    auto* data = alloc.allocate_bytes(1000, alignof(double));

    // 3. 构造对象
    auto* s = alloc.new_object<std::pmr::string>("PMR is awesome!");

    std::cout << *s << '\n';

    alloc.delete_object(s);
    alloc.deallocate_bytes(data, 1000, alignof(double));
}
```

---

### 4. 为什么这些增强很重要？

- **更灵活**：`polymorphic_allocator<>` 成为真正的“通用分配器”。
- **更安全**：`new_object` / `delete_object` 自动处理构造/析构，减少错误。
- **更高性能**：配合 `monotonic_buffer_resource`、`synchronized_pool_resource` 等使用时非常高效。
- **代码更简洁**：减少手动 `allocate` + `construct` 的繁琐操作。

---

### 5. 与 C++17 的主要差异总结

| 特性          | C++17             | C++20                                         |
| ------------- | ----------------- | --------------------------------------------- |
| 默认模板参数  | `T`（必须指定）   | `std::byte`（推荐 `polymorphic_allocator<>`） |
| 原始内存分配  | 需手动 `allocate` | `allocate_bytes` / `deallocate_bytes`         |
| 对象分配+构造 | 较繁琐            | `new_object` / `delete_object`（便捷）        |
| 作为通用类型  | 一般              | **优秀**（vocabulary type）                   |

---

**推荐用法**：
在新代码中，**优先使用** `std::pmr::polymorphic_allocator<>`（空模板） + `new_object` / `delete_object`。

需要我继续讲解 `std::pmr::memory_resource` 的各种实现（如 `monotonic_buffer_resource`、`synchronized_pool_resource`）或实际项目中的高级用法吗？