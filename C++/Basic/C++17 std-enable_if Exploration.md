**`std::enable_if` Exploration** — Complete Guide

`std::enable_if` is one of the most important tools in C++ template metaprogramming, especially before C++20 Concepts.

### What is `std::enable_if`?

- **Header**: `<type_traits>`
- **Purpose**: Enable or disable a function/template overload **at compile time** based on a condition (SFINAE — Substitution Failure Is Not An Error).

It allows you to write **conditional compilation** for templates.

### Basic Form

```cpp
template <bool B, typename T = void>
struct enable_if {
    // no 'type' member if B == false
};

template <typename T>
struct enable_if<true, T> {
    using type = T;
};

// Convenience alias (C++14+)
template <bool B, typename T = void>
using enable_if_t = typename enable_if<B, T>::type;
```

---

### Core Usage Patterns

#### 1. **Basic Function Overloading**

```cpp
#include <type_traits>
#include <iostream>

// Only enabled for integral types
template <typename T>
std::enable_if_t<std::is_integral_v<T>, void>
print(T value) {
    std::cout << "Integral: " << value << '\n';
}

// Only enabled for floating point types
template <typename T>
std::enable_if_t<std::is_floating_point_v<T>, void>
print(T value) {
    std::cout << "Floating: " << value << '\n';
}

int main() {
    print(42);           // calls integral version
    print(3.14);         // calls floating version
    // print("hello");   // compile error - no matching overload
}
```

#### 2. **Return Type Version** (most common)

```cpp
template <typename T>
typename std::enable_if<std::is_integral_v<T>, T>::type
twice(T x) {
    return x * 2;
}

// Modern C++14 style (cleaner)
template <typename T>
std::enable_if_t<std::is_integral_v<T>, T>
twice(T x) {
    return x * 2;
}
```

#### 3. **Default Template Parameters** (very popular)

```cpp
template <typename T, 
          typename = std::enable_if_t<std::is_integral_v<T>>>
void process(T value) {
    // only integral types allowed
}
```

#### 4. **Multiple Conditions**

```cpp
template <typename T>
std::enable_if_t<
    std::is_integral_v<T> && 
    !std::is_same_v<T, bool>,   // exclude bool if desired
    T
>
process(T x);
```

---

### Advanced & Real-World Examples

**Example 1: Perfect Forwarding + Constraints**

```cpp
template <typename T>
std::enable_if_t<std::is_constructible_v<std::string, T>, void>
set_name(T&& name) {
    m_name = std::forward<T>(name);
}
```

**Example 2: Size-based Optimization**

```cpp
template <typename T>
std::enable_if_t<sizeof(T) <= 8, void>
fast_copy(T* dest, const T* src) {
    // use memcpy or direct assignment
}

template <typename T>
std::enable_if_t<sizeof(T) > 8, void>
fast_copy(T* dest, const T* src) {
    // use loop or other strategy
}
```

**Example 3: Tag Dispatching Alternative**

```cpp
template <typename T>
void serialize(const T& obj, std::enable_if_t<std::is_pod_v<T>>* = nullptr) {
    // POD version (fast)
}

template <typename T>
void serialize(const T& obj, std::enable_if_t<!std::is_pod_v<T>>* = nullptr) {
    // Complex type version
}
```

---

### Modern Alternatives (C++20+)

With **Concepts**, `std::enable_if` becomes much less necessary:

```cpp
// Old way
template <typename T>
std::enable_if_t<std::is_integral_v<T>, T> foo(T x);

// Modern way (preferred)
template <std::integral T>
T foo(T x);
```

However, `std::enable_if` is still useful for:
- Supporting older compilers
- Very complex conditions
- Library authors maintaining backward compatibility

---

### Common Pitfalls

1. **Forgetting `::type`** (pre C++14) — very common error.
2. Using it when `if constexpr` would be better (C++17).
3. Overusing it — can make code hard to read.
4. **SFINAE vs Hard Error**: `enable_if` produces substitution failure (clean), not hard error.

---

Would you like me to explore any specific area deeper?

- `std::enable_if` with `std::void_t` (detect idiom)
- Interaction with `std::conditional`
- Complex multi-condition examples
- Comparison with Concepts
- Real project patterns

Just tell me what direction you want to go!

```c++

#include <type_traits>
#include <iostream>

// Only enabled for integral types
template <typename T>
std::enable_if_t<std::is_integral_v<T>, void>
print(T value) {
    std::cout << "Integral: " << value << '\n';
}

// C++26 concepts version ,modern way
template <std::integral T>
void print(T value) {
    std::cout << "Integral (concept): " << value << '\n';
}

// Only enabled for floating point types
template <typename T>
std::enable_if_t<std::is_floating_point_v<T>, void>
print(T value) {
    std::cout << "Floating: " << value << '\n';
}

//only for std::string
template <typename T>
std::enable_if_t<std::is_same_v<T, std::string>, void>
print(T value) {
    std::cout << "String: " << value << '\n';
}

//only for const char*
template <typename T>
std::enable_if_t<std::is_same_v<T, const char*>, void>
print(T value) {
    std::cout << "C-String: " << value << '\n';
}

int main() {
    print(42);           // calls integral version
    print(3.14);         // calls floating version
    print(std::string("hello"));  // calls string version
    print("hello");      // calls C-String version
}   

```

