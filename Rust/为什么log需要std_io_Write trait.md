为什么log需要std::io::Write trait？

因为 `writeln!` 宏并不是 `File` 自带的方法。它实际上需要 `std::io::Write` trait 提供的 `write_fmt()` 能力。

你的代码中：

```rust
writeln!(
    log,
    "hello world :{}",
    now.format("%Y-%m-%d %H:%M:%S")
).unwrap();
```

这里的 `log` 类型是：

```rust
std::sync::MutexGuard<'_, File>
```

`writeln!` 最终需要调用类似：

```rust
log.write_fmt(...)
```

而 `write_fmt()` 来自：

```rust
std::io::Write
```

所以：

```rust
use std::io::Write;
```

不是为了导入一个普通函数，而是为了把 `Write` trait 引入当前作用域，让 Rust 能进行 **trait method resolution（trait 方法解析）**。

你的代码实际上可以理解成：

```rust
use std::io::Write;

let mut log = my_log.lock().unwrap();

log.write_fmt(format_args!(
    "hello world :{}\n",
    now.format("%Y-%m-%d %H:%M:%S")
))?;
```

其中 `write_fmt()` 是 `Write` trait 提供的：

```rust
pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;

    fn write_fmt(&mut self, args: fmt::Arguments<'_>) -> Result<()>;
}
```

而 `File` 实现了 `Write`：

```text
std::fs::File
     │
     └── impl std::io::Write
              │
              ├── write()
              ├── flush()
              └── write_fmt()
```

同时这里还有一个容易忽略的地方：

```rust
let mut log = my_log.lock().unwrap();
```

`log` 并不是 `File` 本身，而是：

```rust
MutexGuard<File>
```

但是 Rust 会通过 `Deref/DerefMut` 找到里面的 `File`，再找到 `File` 对 `Write` 的实现，因此：

```rust
writeln!(log, ...)
```

可以正常工作。

所以你可以把这件事情简单记成：

```text
writeln!(file, ...)
       │
       ▼
需要 Write trait
       │
       ▼
std::io::Write
       │
       ▼
File implements Write
```

这也是 Rust 中经常出现的一种情况：

```rust
use std::io::Read;
use std::io::Write;
use std::io::Seek;
```

这些 `use` 有时候看起来“什么都没导入”，但实际上是在**把 trait 放入作用域，以便调用 trait 提供的方法**。

例如：

```rust
use std::io::Write;

let mut file = File::create("foo.txt")?;
file.write_all(b"hello")?;
```

如果去掉：

```rust
use std::io::Write;
```

通常就会出现类似：

```text
no method named `write_all` found for struct `File`
```

并且编译器一般会进一步提示：

```text
trait `Write` which provides `write_all` is implemented
but not in scope
```

这也是 Rust 和 C++ 一个比较明显的区别：**Rust 的 trait 方法通常需要 trait 在当前作用域可见，才能进行这种方法调用。**