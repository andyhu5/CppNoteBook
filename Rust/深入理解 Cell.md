深入理解 Cell<T>

关键点是：

> `Cell<T>` 允许你在**不需要 `&mut` 的情况下，修改内部的 `T`**。

先看完整代码：

```rust
use std::cell::Cell;

struct Person {
    name: String,
    age: u32,
}

fn main() {
    let wang = Person {
        name: "Alice".to_string(),
        age: 30,
    };

    let wang = Cell::new(Some(wang));

    // 修改内部值
    wang.set(Some(Person {
        name: "Bob".to_string(),
        age: 25,
    }));

    // 临时取出 Person
    let p = wang.replace(None).unwrap();

    println!("Name: {}, Age: {}", p.name, p.age);

    // 放回去
    wang.set(Some(p));
}
```



```rust
//使用RefCell

use std::cell::RefCell;

#[derive(Clone)]
struct Person {
    name: String,
    age: u32,
}

fn main() {
    let wang = RefCell::new(Person {
        name: "Alice".to_string(),
        age: 30,
    });
    
    // 修改内部值
    *wang.borrow_mut() = Person {
        name: "Bob".to_string(),
        age: 25,
    };

    // 通过 borrow() 获取只读引用
    let p = wang.borrow();
    println!("Name: {}, Age: {}", p.name, p.age);
    // p 在这里自动释放借用
}
```

这里最容易困惑的是这一句：

```rust
let wang = Cell::new(Some(wang));
```

它实际上把原来的：

```text
Person
```

变成了：

```text
Cell<Option<Person>>
```

也就是：

```text
wang
 │
 ▼
Cell
 │
 ▼
Option<Person>
 │
 ▼
Person
```

为什么外面要包一个 `Option`？因为 `Cell<T>` 对于非 `Copy` 类型不能直接通过 `get()` 把值拿出来。

------

### 1. `Cell::new`

最开始：

```rust
let wang = Person {
    name: "Alice".to_string(),
    age: 30,
};
```

此时：

```text
wang → Person("Alice", 30)
```

然后：

```rust
let wang = Cell::new(Some(wang));
```

注意这里的 `let wang` 是**变量遮蔽（shadowing）**。

原来的 `wang`：

```text
Person
```

被新的 `wang` 遮蔽了：

```text
Cell<Option<Person>>
```

因此现在：

```rust
wang.set(...)
```

调用的是 `Cell` 的方法。

------

### 2. `set()`：直接替换内部值

```rust
wang.set(Some(Person {
    name: "Bob".to_string(),
    age: 25,
}));
```

原来：

```text
Cell
 └── Some(Person("Alice", 30))
```

执行 `set()` 后：

```text
Cell
 └── Some(Person("Bob", 25))
```

重要的是：

```rust
set()
```

**不会返回旧值。**

所以原来的：

```text
Person("Alice", 30)
```

会被丢弃（drop）。

因此注释：

```rust
// previous one will be dropped
```

是准确的。

可以把它粗略理解成：

```cpp
Cell<T>
```

内部类似维护一个 `T`，然后：

```rust
cell.set(new_value);
```

相当于“用新值替换旧值”。

但 Rust 的 `Cell` 通过类型系统保证了这种操作的安全性。

------

### 3. 为什么不能直接 `get()`？

这里的 `Person`：

```rust
struct Person {
    name: String,
    age: u32,
}
```

不是 `Copy` 类型。

因为：

```rust
String
```

拥有堆内存。

如果写：

```rust
let p = wang.get();
```

编译器会拒绝。

原因是 `get()` 要把里面的 `T` **复制出来**，而：

```rust
Person
```

不能简单复制。

如果 `T` 是 `Copy`：

```rust
let x = Cell::new(100);

let y = x.get();
```

就可以。

因为：

```text
i32
```

实现了 `Copy`。

------

### 4. 所以这里使用 `replace()`

代码：

```rust
let p = wang.replace(None).unwrap();
```

是整个例子的核心。

`replace()` 的语义是：

> 用一个新值替换 `Cell` 中的旧值，并把旧值返回。

原来：

```text
Cell
 └── Some(Person("Bob", 25))
```

执行：

```rust
wang.replace(None)
```

之后：

```text
Cell
 └── None
```

同时：

```rust
p
```

得到：

```text
Person("Bob", 25)
```

所以可以理解成：

```text
之前：

wang ──→ Cell ──→ Some(Person)
                    │
                    └── Bob, 25


replace(None)


之后：

wang ──→ Cell ──→ None

p ──→ Person
      Bob, 25
```

这就是为什么使用 `Option<Person>`。

因为它允许我们暂时把 `Person` 从 `Cell` 中**完整地移动（move）出来**。

------

### 5. `unwrap()`

这里：

```rust
let p = wang.replace(None).unwrap();
```

`replace(None)` 返回的是：

```rust
Option<Person>
```

因为 `Cell` 原来可能是：

```rust
Some(Person)
```

也可能是：

```rust
None
```

所以需要：

```rust
.unwrap()
```

把：

```text
Some(Person)
```

转换成：

```text
Person
```

如果实际是：

```text
None
```

则 `unwrap()` 会 panic。

------

### 6. 为什么最后又 `set(p)`？

取出来以后：

```rust
let p = wang.replace(None).unwrap();
```

`Cell` 已经变成：

```text
None
```

而 `p` 拥有这个 `Person`：

```text
p → Person("Bob", 25)
```

打印：

```rust
println!("Name: {}, Age: {}", p.name, p.age);
```

然后：

```rust
wang.set(Some(p));
```

把 `p` **移动回 Cell**：

```text
之前：

wang → Cell → None

p → Person("Bob", 25)


set(Some(p))


之后：

wang → Cell → Some(Person("Bob", 25))
```

注意：

```rust
set(Some(p))
```

之后，`p` 就不能再使用了。

因为 `Person` 的所有权已经移动回 `Cell`。

------

### 7. 整个过程其实就是“取出 → 使用 → 放回”

可以浓缩成：

```rust
let p = wang.replace(None).unwrap();

println!("Name: {}, Age: {}", p.name, p.age);

wang.set(Some(p));
```

也就是：

```text
Cell<Option<Person>>
       │
       │ replace(None)
       ▼
     Person
       │
       │ 使用
       ▼
     Person
       │
       │ set(Some(p))
       ▼
Cell<Option<Person>>
```

这种模式非常重要。

------

### 8. 为什么不直接使用 `RefCell`？

这正好可以和你上一问联系起来。

`Cell`：

```rust
Cell<T>
```

倾向于：

```text
把整个 T 拿出来 / 替换整个 T
```

而 `RefCell`：

```rust
RefCell<T>
```

倾向于：

```text
借用里面的 T
```

例如：

```rust
use std::cell::RefCell;

let wang = RefCell::new(Person {
    name: "Bob".to_string(),
    age: 25,
});

let p = wang.borrow();

println!("{}", p.name);
```

这里没有把 `Person` 从 `RefCell` 中移动出来，而是得到一个：

```text
Ref<Person>
```

相当于 Rust 在运行时管理一个借用。

所以可以记成：

```text
Cell<T>
    → 适合“整体替换 / 整体取出”

RefCell<T>
    → 适合“借用内部对象进行操作”
```

从 C++ 的角度，`Cell` 最值得记住的不是“它是一个容器”，而是：

> **`Cell<T>` 是 Rust 实现 interior mutability 的一种工具；它允许通过 `&self` 修改/替换内部值，同时不允许你获得指向内部值的普通 `&mut T` 引用。**

而 `RefCell<T>` 则进一步提供了：

```text
borrow()      → 运行时共享借用
borrow_mut()  → 运行时独占借用
```

这两个放在一起理解，会比单独记 API 更容易。