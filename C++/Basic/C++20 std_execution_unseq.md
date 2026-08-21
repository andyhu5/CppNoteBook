**C++20 `std::execution::unseq` 完整介绍**

`std::execution::unseq` 是 C++20 在 `<execution>` 头文件中新增的**执行策略**（Execution Policy）。

---

### 1. 含义

**unseq = Unsequenced**（非序列化 / 无序执行）

它表示算法的执行是**单线程的**（sequential），但**允许操作被任意交错（interleaved）和重排序**，以便编译器更好地进行**向量化（SIMD）**优化。

---

### 2. 与其他执行策略的对比

| 执行策略                    | C++版本   | 是否并行 | 是否允许重排序/交错 | 主要用途           |
| --------------------------- | --------- | -------- | ------------------- | ------------------ |
| `std::execution::seq`       | C++17     | 否       | 否（严格顺序）      | 默认安全顺序执行   |
| `std::execution::unseq`     | **C++20** | **否**   | **是**（强允许）    | **向量化（SIMD）** |
| `std::execution::par`       | C++17     | 是       | 否                  | 多线程并行         |
| `std::execution::par_unseq` | C++17     | 是       | 是                  | 并行 + 向量化      |

**关键区别**：
- `unseq` **不使用多线程**，只在一个线程内执行，但**放松了操作的顺序约束**，允许编译器生成 SIMD 指令（如 AVX2、NEON 等）。

---

### 3. 使用示例

```cpp
#include <execution>
#include <vector>
#include <algorithm>
#include <iostream>

int main() {
    std::vector<double> v(1'000'000);
    // 填充数据...

    // 使用 unseq 执行策略
    std::for_each(std::execution::unseq, v.begin(), v.end(), 
        [](double& x) {
            x = std::sin(x) * std::cos(x);   // 适合向量化
        });

    // 或用于 transform
    std::vector<double> result(v.size());
    std::transform(std::execution::unseq,
                   v.begin(), v.end(), result.begin(),
                   [](double x) { return x * x + 1.0; });
}
```

---

### 4. 什么时候使用 `unseq`？

**推荐场景**：
- 计算密集型循环，且每个元素操作**相互独立**（无数据依赖）
- 希望编译器进行**自动向量化**（SIMD）
- 不希望引入多线程的复杂性（避免数据竞争、同步开销）
- 处理大数组的数学运算、图像处理、信号处理等

**不适合场景**：
- 操作之间有依赖（如累加、前缀和）
- 可能抛出异常的操作（`unseq` 对异常处理有严格限制）
- 需要严格顺序的操作（如输出顺序敏感）

---

### 5. 重要注意事项

1. **未定义行为风险**  
   如果你的 lambda 中存在**副作用**（如修改全局变量、通过指针访问共享内存等），使用 `unseq` 可能导致未定义行为。

2. **异常处理**  
   如果 functor 抛出异常，程序会调用 `std::terminate()`。

3. **编译器支持**  
   需要编译器开启向量化优化（通常 `-O3` + 对应 SIMD flag）。不同编译器（GCC、Clang、MSVC）对 `unseq` 的向量化效果差异较大。

4. **与 `par_unseq` 的关系**  
   `par_unseq` = 并行 + unseq  
   `unseq` = 单线程版 `par_unseq`

---

### 6. 总结

- `std::execution::unseq` 是 C++20 为**单线程向量化**专门设计的执行策略。
- 它在**不引入多线程**的情况下，给了编译器更大的优化空间。
- 在高性能计算、数值算法中非常实用，是 C++20 执行策略体系的重要补充。

---

**推荐写法**（现代风格）：

```cpp
std::for_each(std::execution::unseq, begin, end, func);
```

需要我给出更多实际性能对比示例，或者讲解 `unseq` 在自定义迭代器上的使用吗？