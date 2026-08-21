## 不用friend和用friend有何区别?

```c
class Vector {
private:
    double x, y;

public:
    Vector(double x = 0, double y = 0) : x(x), y(y) {}

    // 非成员函数运算符
    /*friend*/ Vector operator+(const Vector& b) {
        return Vector(x + b.x, y + b.y);
    }

    friend std::ostream& operator<<(std::ostream& os, const Vector& v) {
        return os << "(" << v.x << ", " << v.y << ")";
    }
};

int main() {
    Vector v1(1.0, 2.0);
    Vector v2(3.0, 4.0);
    Vector v3 = v1 + v2; // 使用非成员函数 operator+
    std::cout << "v3: " << v3 << std::endl; // 输出: v3: (4, 6)
    return 0;
}
这种不用friend和用friend有何区别？
```



**很好的问题！** 这两种写法有**本质区别**：

### 1. 当前代码（**不用 `friend`**）

```cpp
Vector operator+(const Vector& b) {   // 成员函数
    return Vector(x + b.x, y + b.y);
}
```

**这是成员函数（Member Function）**：

- **参数个数**：只有 1 个参数（右操作数）。
- 左操作数隐式是 `*this`。
- 可以直接访问本类的 `private` 成员（`x`、`y`）。
- **调用方式**：`v1 + v2` 实际上等价于 `v1.operator+(v2)`。

### 2. 加上 `friend` 的写法（之前推荐的）

```cpp
friend Vector operator+(const Vector& a, const Vector& b) {  // 非成员函数
    return Vector(a.x + b.x, a.y + b.y);
}
```

**这是非成员函数（Non-member Function）**，通过 `friend` 获得访问私有成员的权限。

---

### 两者主要区别对比

| 方面         | **成员函数**（不用 `friend`）             | **非成员函数**（用 `friend`）                  | 推荐场景               |
| ------------ | ----------------------------------------- | ---------------------------------------------- | ---------------------- |
| 参数数量     | 1 个 (`this` + 右操作数)                  | 2 个 (左操作数 + 右操作数)                     | -                      |
| 左操作数类型 | **必须**是 `Vector`                       | 可以是 `Vector` 或能隐式转换到 `Vector` 的类型 | **非成员更灵活**       |
| 访问私有成员 | 可以（因为是成员）                        | 需要 `friend` 才能访问                         | -                      |
| 对称性       | 不对称（`v1 + v2` 可以，`int + v1` 不行） | 对称（支持混合类型）                           | **+、== 等推荐非成员** |
| 隐式转换     | 左操作数**不能**隐式转换                  | 左右操作数都可以隐式转换                       | 非成员胜出             |
| 可被继承     | 可以被派生类继承/重写                     | 不能继承                                       | -                      |
| 代码风格     | 更像“对象的行为”                          | 更像“两个对象之间的操作”                       | -                      |

### 实际例子说明区别

```cpp
Vector v(1, 2);
Vector result1 = v + 5.0;        // 成员函数：可以（右操作数隐式转换）
Vector result2 = 5.0 + v;        // 成员函数：**错误！** 左操作数不能隐式转换

// 如果用 friend 非成员函数：
Vector result3 = 5.0 + v;        // **可以**成功（左右都支持隐式转换）
```

### 总结推荐

- **`operator+`、`operator-`、`operator*`、`operator==` 等对称运算符**：**强烈推荐使用 `friend` 非成员函数**。
- **`operator=`、`operator[]`、`operator->`、`operator()`** 等：**必须用成员函数**（不能用 friend）。
- `operator<<`、`operator>>`：几乎总是用 **friend 非成员函数**（因为左操作数是 `ostream`）。

---

**你的当前代码（不用 friend）也能正常工作**，但**不够灵活**。  
如果你希望支持 `Vector + double`、`double + Vector` 等混合运算，就应该改成 `friend` 版本。

需要我把**完整推荐版本**（同时支持 `Vector + Vector` 和 `Vector + double`）给你吗？