Rust VS C++ 数据结构

```text
Rust HashMap<K, V>   ≈   C++ std::unordered_map<K, V>
Rust BTreeMap<K, V>  ≈   C++ std::map<K, V>
```

主要区别在于底层数据结构。

| Rust            | C++                       | 底层结构      | Key 是否有序 |
| --------------- | ------------------------- | ------------- | ------------ |
| `HashMap<K,V>`  | `std::unordered_map<K,V>` | 哈希表        | 否           |
| `BTreeMap<K,V>` | `std::map<K,V>`           | 平衡树/B-tree | 是           |

例如 Rust：

```rust
use std::collections::HashMap;

let mut m = HashMap::new();

m.insert("hello", 10);
m.insert("world", 20);

println!("{}", m["hello"]);
```

大致对应 C++：

```cpp
#include <unordered_map>

std::unordered_map<std::string, int> m;

m["hello"] = 10;
m["world"] = 20;

std::cout << m["hello"];
```

而如果你想要 Rust 中类似 C++ `std::map` 的行为：

```rust
use std::collections::BTreeMap;

let mut m = BTreeMap::new();

m.insert("hello", 10);
m.insert("world", 20);
```

这里的 `BTreeMap` 会按照 key 排序。

一个很重要的区别是 Rust 标准库没有直接叫 `TreeMap` 的类型，而是：

```text
HashMap   → 哈希表
BTreeMap  → B-tree
```

另外，`BTreeMap` 和 C++ `std::map` 虽然用途非常接近，但底层实现也不是完全一样：

```text
C++ std::map
    ↓
通常是红黑树等平衡二叉搜索树

Rust BTreeMap
    ↓
B-tree
```

因此，如果你是在建立 **C++ → Rust 容器对应关系**，可以先记住：

```text
std::vector        → Vec
std::deque         → VecDeque
std::list          → LinkedList
std::array         → [T; N]
std::unordered_map → HashMap
std::map           → BTreeMap
std::set           → BTreeSet
std::unordered_set → HashSet
```

尤其是 `HashMap` 和 `BTreeMap`，这个区别在 Rust 里非常重要。