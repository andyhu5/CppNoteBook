**是的，`friend` 在模板类（template class）中的使用原则基本一致，但写法和细节上有一些**重要差异**。**

下面用一个**模板版本的 `Vector`** 来详细说明：

### 1. 模板类中 `friend` 的两种写法对比

```cpp
#include <iostream>

template<typename T>
class Vector {
private:
    T x, y;

public:
    Vector(T x = 0, T y = 0) : x(x), y(y) {}

    // ==================== 写法1：成员函数（不用 friend） ====================
    Vector operator+(const Vector& b) const {   // 注意 const
        return Vector(x + b.x, y + b.y);
    }

    // ==================== 写法2：非成员函数（推荐使用 friend） ====================
    // 正确写法（模板类中最常见）
    template<typename U>
    friend Vector<U> operator+(const Vector<U>& a, const Vector<U>& b);

    // 输出运算符
    friend std::ostream& operator<<(std::ostream& os, const Vector& v) {
        return os << "(" << v.x << ", " << v.y << ")";
    }
};

// 在类外定义 friend 声明的 operator+
template<typename T>
Vector<T> operator+(const Vector<T>& a, const Vector<T>& b) {
    return Vector<T>(a.x + b.x, a.y + b.y);
}

int main() {
    Vector<double> v1(1.0, 2.0);
    Vector<double> v2(3.0, 4.0);
    Vector<double> v3 = v1 + v2;

    std::cout << "v3: " << v3 << std::endl;
}
```

### 2. 关键区别与注意事项

| 方面                                 | 普通类 (`class Vector`)       | 模板类 (`template<typename T> class Vector`)     |
| ------------------------------------ | ----------------------------- | ------------------------------------------------ |
| 成员函数 `operator+`                 | 直接写即可                    | 直接写即可（推荐加 `const`）                     |
| 非成员 `friend operator+`            | `friend Vector operator+(..)` | 需要 `template<typename U>` + `friend Vector<U>` |
| `friend` 声明位置                    | 类内直接声明                  | 类内声明时通常需要额外 `template<>` 声明         |
| 类外定义                             | 可以直接定义                  | 必须在类外提供模板定义                           |
| 支持混合类型（如 `Vector + double`） | 较难                          | 更容易（通过额外重载）                           |

### 3. 更灵活的 `friend` 写法（推荐）

如果你希望 `operator+` 支持**不同类型**的运算（如 `Vector<double> + Vector<int>`），可以这样写：

```cpp
template<typename T>
class Vector {
    // ...
    
    // 让所有 Vector<U> 都是当前 Vector<T> 的 friend
    template<typename U>
    friend class Vector;        // （较少用）

    // 或者只让 operator+ 成为 friend
    template<typename U>
    friend Vector<U> operator+(const Vector<U>& a, const Vector<U>& b);
};
```

### 总结回答你的问题：

- **核心原理一样**：`friend` 都是为了让**非成员函数**能访问私有成员。
- **模板中更复杂**：必须处理**模板参数**，声明时通常需要多写一行 `template<typename U>`。
- **强烈推荐**在模板类中使用 **非成员 `friend`** 写法，尤其是运算符重载。
- **成员函数版本** 在模板中也可以正常工作，但**灵活性较差**（不支持左操作数隐式转换）。

---

**想看哪种更详细的版本？**

1. 支持 `Vector<T> + Vector<U>` 混合运算的完整示例
2. `friend` 模板函数的各种写法对比
3. 把 `operator<<` 也写成模板形式

告诉我你的需求，我立刻给你对应代码！