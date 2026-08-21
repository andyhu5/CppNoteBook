**`compare_exchange_strong` 详细解释**（以 `std::atomic<double>` 为例）

这是原子操作中最重要、最强大的操作之一，常简称为 **CAS**（Compare And Swap）。

### 1. 完整调用形式

```cpp
value.compare_exchange_strong(expected, desired);
```

#### 参数说明：

| 参数       | 类型                 | 含义                                         | 是否会被修改   |
| ---------- | -------------------- | -------------------------------------------- | -------------- |
| `expected` | `double&`（引用）    | **期望值**（当前我认为原子变量应该是什么值） | **可能被修改** |
| `desired`  | `double`（按值传递） | **想要设置的新值**                           | 不修改         |

---

### 2. 执行逻辑（非常重要）

`compare_exchange_strong` **原子地** 执行以下操作：

1. **比较** `value` 当前的实际值 是否 **等于** `expected`。
2. **如果相等**：
   - 把 `value` 修改为 `desired`。
   - 返回 `true`。
3. **如果不相等**：
   - **不修改** `value`。
   - 把 `value` 的**当前真实值** 写入 `expected`（更新 expected）。
   - 返回 `false`。

**这整个过程是原子完成的**，不会被其他线程打断。

---

### 3. 代码示例 + 详细行为

```c
#include <atomic>
#include <iostream>

int main() {
    std::atomic<double> value{0.0};

    value.store(1.5);
    value.fetch_add(2.7);           // 原子加
    value += 0.8;                   // 等价写法

    double current = value.load();
    // std::cout << current << '\n';   // 5.0

    // CAS 操作也支持
    double expected = 5.0;
    double desired = 10.0;
    if(value.compare_exchange_strong(expected, desired)) {
        std::cout << "Exchange successful: " << expected << " - " << desired <<" - "<<  value.load() << "\n";
    } else {
        std::cout << "Exchange failed: " << expected << " - " << desired << " - " << value.load() << "\n";
    }
}
```



```cpp
std::atomic<double> value{5.0};

double expected = 5.0;
double desired = 10.0;

bool success = value.compare_exchange_strong(expected, desired);

if (success) {
    std::cout << "成功！value 已被改为 " << value.load() << "\n";
} else {
    std::cout << "失败！当前实际值是 " << expected << "\n";
}
```

**可能的结果**：

- **成功情况**（最常见）：
  - `value` 原来确实是 `5.0` → 改为 `10.0`，返回 `true`

- **失败情况**（被其他线程抢先修改）：
  - 假如其他线程把 `value` 改成了 `7.0`
  - 则 `compare_exchange_strong` 失败，返回 `false`，同时 `expected` 被更新为 `7.0`

---

### 4. 返回值

- **`true`**：比较成功，交换已发生（value 已被设为 desired）。
- **`false`**：比较失败，value 未被修改，但 `expected` 已更新为最新值。

---

### 5. `expected` 参数被修改的作用（核心设计）

这是 CAS 操作最精妙的地方！

失败时自动把 `expected` 更新为最新值，**方便你立即重试**：

```cpp
double expected = 5.0;
double desired = 10.0;

while (!value.compare_exchange_strong(expected, desired)) {
    // expected 已经被自动更新为最新值
    // 可以在这里做一些计算，或者直接重试
}
```

这种“失败时自动加载最新值”的设计极大简化了**无锁编程**。

---

### 6. `compare_exchange_strong` vs `compare_exchange_weak`

| 特性                         | `strong`         | `weak`                                 |
| ---------------------------- | ---------------- | -------------------------------------- |
| 虚假失败（spurious failure） | **不会**发生     | 可能发生（即使值相同也可能返回 false） |
| 性能                         | 稍慢             | **更快**（尤其在循环中使用）           |
| 推荐使用场景                 | 简单、非循环场景 | **循环中重试**（推荐）                 |

**规则**：
- 单次尝试 → 用 `strong`
- 放在 `while` 循环中 → 优先用 `weak`（性能更好）

---

### 7. 完整推荐写法（C++20）

```cpp
std::atomic<double> value{5.0};

double expected = value.load();   // 先读取当前值
double desired = expected + 3.14;

while (!value.compare_exchange_weak(expected, desired)) {
    // expected 已更新，可选择继续计算 desired
    desired = expected + 3.14;
}
```

---

**总结**：

- `expected`：我期望的值（会被修改）
- `desired`：我想设置的新值
- 返回 `true` = 成功交换
- 返回 `false` = 失败，且 `expected` 被更新为当前最新值

这是实现**无锁数据结构**（lock-free）的核心原语。

需要我继续讲解 `compare_exchange_weak` 的细节、ABA 问题、或者实际的无锁队列/计数器例子吗？