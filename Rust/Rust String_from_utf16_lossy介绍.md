`lossy` 的意思是：**“有损转换”**。

这里：

```rust
let hostname = String::from_utf16_lossy(&buffer);
```

表示：

> 尝试把 UTF-16 数据转换成 Rust 的 `String`（UTF-8）。如果 UTF-16 中存在无法正确组成合法 Unicode 字符的数据，就用 `�`（U+FFFD，replacement character）替代，而不是返回错误。

对比一下 Rust 提供的两个 API：

```rust
String::from_utf16(&buffer)
```

这是**严格转换**：

```rust
Result<String, FromUtf16Error>
```

如果 UTF-16 非法：

```text
Err(...)
```

而：

```rust
String::from_utf16_lossy(&buffer)
```

是**宽松转换**：

```text
String
```

遇到非法 UTF-16 时直接替换成：

```text
�
```

例如：

```rust
let data = vec![0x0048, 0x0069];

let s = String::from_utf16_lossy(&data);

println!("{}", s);
```

结果：

```text
Hi
```

如果存在非法 UTF-16：

```rust
let data = vec![
    0x0048, // H
    0x0069, // i
    0xD800, // 孤立的 surrogate，非法
];

let s = String::from_utf16_lossy(&data);

println!("{}", s);
```

结果类似：

```text
Hi�
```

所以：

```text
from_utf16()
        │
        ├── 合法 → Ok(String)
        └── 非法 → Err

from_utf16_lossy()
        │
        ├── 合法 → String
        └── 非法 → 用 � 替换后继续
```

在你的 Windows API 场景中：

```rust
let hostname = String::from_utf16_lossy(&buffer);
```

通常是可以接受的，因为 `GetComputerNameW` 返回的是 Windows 的合法 UTF-16 字符串。

如果你希望**严格处理错误**，可以写：

```rust
let hostname = String::from_utf16(&buffer)?;
```

当然，这要求你的 `main` 或外围函数能够处理 `Result`。

因此这里的 `lossy` 可以直接理解为：

> **转换过程中如果遇到无法表示的数据，不报错，丢失/替换掉有问题的数据。**