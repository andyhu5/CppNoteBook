## 

### 1. **模块名默认等于文件名/目录名**
是的，默认情况下：
- `foo.rs` → 模块名是 `foo`
- `libs/` 目录 → 模块名是 `libs`

### 2. **但你可以改变模块名（通过重命名）**

你可以使用 `mod` 关键字重命名导入的模块：

**方法：使用 `as` 重命名**

```rust
// src/main.rs

// 或者重命名整个 libs 模块
mod libx {
    pub mod foo;
    pub mod itools;
    pub mod stringsutils;
}

// 或者重命名整个 libs 模块
use libx as utils;

fn main() {
    // 使用重命名后的名称
    utils::foo::some_function();
}
```

### 3. **在你的情况下**

你把文件移到了 `libs` 目录，但你的 `main.rs` 中还是：
```rust
mod foo;  // ❌ 错误！foo.rs 不在 src 根目录了
mod itools;
mod stringsutils;
```

所以你需要：

**选项A：保持原模块名（推荐）**
```rust
// src/libs/mod.rs
pub mod foo;
pub mod itools;
pub mod stringsutils;

// src/main.rs
mod libs;  // 模块名是 libs，包含 foo, itools, stringsutils

fn main() {
    libs::foo::some_function();  // 通过 libs 访问
    // 或者
    use libs::foo;
    foo::some_function();
}
```

**选项B：重新导出，保持原有访问方式**
```rust
// src/libs/mod.rs
pub mod foo;
pub mod itools;
pub mod stringsutils;

// src/main.rs
mod libs;

// 将子模块重新导出到根级别
use libs::foo;
use libs::itools;
use libs::stringsutils;

fn main() {
    // 现在可以直接用原来的名字
    foo::some_function();
    itools::some_function();
    stringsutils::some_function();
}
```


## 总结

- **默认情况**：模块名 = 文件名/目录名
- **可以改变**：通过 `as` 重命名，或通过重新导出
- **你的情况**：移动目录后，需要相应地改变模块声明方式，不一定非要改模块名，但访问路径会变化

**最简单的做法**：创建 `src/libs/mod.rs`，然后在 `main.rs` 中用 `mod libs;`，通过 `libs::foo` 来访问。这样你不需要改模块名，只是访问路径多了一层 `libs::`。