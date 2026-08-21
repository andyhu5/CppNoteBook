**是的，模板类中除了 `operator+` 这种非成员运算符之外，还有很多场景需要额外使用 `template <typename U>`（或 `typename V` 等）。**

下面系统性地列出**最常见的场景**，并给出代码示例：

### 1. **非成员友元函数（最常见）** —— 如 `operator+`、`operator==` 等

这是你目前遇到的情况：

```cpp
template<typename T>
class Vector {
private:
    T x, y;

public:
    // 声明一个独立的模板友元函数
    template<typename U>                    // ← 这里必须加
    friend Vector<U> operator+(const Vector<U>& a, const Vector<U>& b);
};

// 类外定义
template<typename T>
Vector<T> operator+(const Vector<T>& a, const Vector<T>& b) { ... }
```

### 2. **让其他模板类成为友元**

```cpp
template<typename T>
class Vector {
    template<typename U>                    // ← 必须加
    friend class Matrix;                    // 所有 Matrix<U> 都是 Vector<T> 的友元
};
```

### 3. **友元为其他模板类的特定成员函数**

```cpp
template<typename T>
class Vector {
    template<typename U>                    // ← 必须加
    friend void Matrix<U>::multiply(const Vector<T>& v);  // 只让 Matrix 的特定成员成为友元
};
```

### 4. **支持混合类型运算（不同模板参数）**

这是实际开发中非常实用的场景：

```cpp
template<typename T>
class Vector {
public:
    // 支持 Vector<T> + Vector<U>
    template<typename U>                                 // ← 这里必须加
    friend Vector<decltype(T{} + U{})> operator+(const Vector<T>& a, const Vector<U>& b);
};
```

### 5. **友元为普通模板函数（非运算符）**

```cpp
template<typename T>
class Vector {
    // 声明一个通用模板函数为友元
    template<typename U, typename V>                     // ← 必须加
    friend void printPair(const Vector<U>& u, const Vector<V>& v);
};
```

### 6. **友元为特化版本或部分特化**

```cpp
template<typename T>
class Vector {
    // 只让 Vector<int> 的某些函数成为友元
    friend void specialFunc(Vector<int>& v);   // 不需要额外 template<>
};
```

### 7. **模板类的模板构造函数 / 赋值运算符**

```cpp
template<typename T>
class Vector {
public:
    // 拷贝/移动构造函数支持不同类型
    template<typename U>                    // ← 必须加
    Vector(const Vector<U>& other);
};
```

---

### 总结：什么时候需要在模板类里写 `template <typename U>`？

当你想在**模板类内部**声明以下内容时，通常需要额外写一行 `template<...>`：

- 非成员友元函数（`friend` + 自由函数）
- 其他模板类作为友元
- 其他模板类的成员函数作为友元
- 支持**不同模板参数**的运算/转换（混合类型）
- 模板化的构造函数、赋值运算符等

**记忆口诀**：
> **只要友元函数/类本身是模板的（或需要支持不同类型），就必须在 `friend` 声明前面加上 `template<typename U>`。**

---

**你现在最想看哪一个完整示例？**

1. 支持 `Vector<T> + Vector<U>`（不同类型相加）
2. 完整的 `Vector` 模板类 + 多个友元运算符
3. `friend` 模板类的实际例子（如 `Vector` 和 `Matrix` 互相友元）

告诉我具体需求，我马上给你可编译的完整代码！