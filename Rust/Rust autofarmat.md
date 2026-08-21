#### 使用 Rust 的 `rust-analyzer`，保存时自动执行 rustfmt

打开 VS Code 的 `settings.json`，加入：

```json
{
    "editor.formatOnSave": true,
    "[rust]": {
        "editor.defaultFormatter": "rust-lang.rust-analyzer"
    }
}
```

如果你只希望 Rust 保存时自动格式化，而其他语言不格式化，推荐这样配置：

```json
{
    "[rust]": {
        "editor.defaultFormatter": "rust-lang.rust-analyzer",
        "editor.formatOnSave": true
    }
}
```

前提是已经安装 VS Code 扩展：

**rust-analyzer**

另外，Rust 的格式化实际上使用的是 `rustfmt`。可以检查：

```powershell
rustfmt --version
```

如果没有安装：

```powershell
rustup component add rustfmt
```

之后你按 `Ctrl+S`，Rust 代码就会自动按照 `rustfmt` 规则格式化。

例如：

```rust
fn main() {
    let x=10;
    println!("x = {}",x);
}
```

保存后会变成：

```rust
fn main() {
    let x = 10;
    println!("x = {}", x);
}
```

如果你使用的是 Cargo 项目，也可以直接手动执行：

```powershell
cargo fmt
```

通常建议 VS Code 用 `formatOnSave`，CI/CD 中再用：

```powershell
cargo fmt --check
```

这样可以检查代码是否符合 `rustfmt`，但不会修改代码。