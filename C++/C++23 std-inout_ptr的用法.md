# C++23 std::inout_ptr的用法

看起来你想用 C++23 引入的 std::inout_ptr 来改写之前的代码。std::inout_ptr 是 C++23 标准库中的一个工具（位于 <memory> 头文件中），专门设计用来与需要输入/输出指针（T**）的 C 风格 API 交互，同时与智能指针（如 std::unique_ptr）配合使用。

在你的例子中，我们的目标是改写以下代码：

cpp

```cpp
auto deleter = [](FILE* fp) { if (fp) fclose(fp); };
std::unique_ptr<FILE, decltype(deleter)> file(fopen("example.txt", "r"), deleter);
```

问题分析

std::inout_ptr 的典型用途是将一个智能指针的资源释放并重新分配给新的资源，通常用于这样的 C 风格函数：

cpp

```cpp
int some_c_function(T** in_out);
```

- T** in_out 是一个输入/输出参数：函数会释放 *in_out 指向的资源（如果有），然后将新的资源地址写入 *in_out。
- std::inout_ptr 会将智能指针的原始指针地址传递给这样的函数，并在调用后自动更新智能指针管理的新资源。

但是，fopen 的签名是 FILE* fopen(const char* filename, const char* mode)，它直接返回一个 FILE*，而不是通过 FILE** 这样的输入/输出参数来操作。这意味着 fopen 并不直接适合 std::inout_ptr，因为 std::inout_ptr 需要一个函数能够通过指针参数修改资源。

为了展示 std::inout_ptr 的用法，我们需要构造一个符合 std::inout_ptr 使用场景的例子。以下我会分两部分处理：

1. 解释为什么 fopen 不适合 std::inout_ptr，并保持原始代码。
2. 构造一个假设的场景，使用 std::inout_ptr 改写类似的操作。

------

1. 为什么 fopen 不适合 std::inout_ptr

fopen 的用法是：

cpp

```cpp
FILE* fp = fopen("example.txt", "r");
```

- fopen 返回一个 FILE*，而不是通过一个 FILE** 参数来修改指针。
- std::inout_ptr 适用于需要 FILE** 参数的函数，比如一个函数会释放旧的 FILE* 并分配一个新的 FILE*。

因此，对于 fopen，原始代码已经是最合适的：

cpp

```cpp
auto deleter = [](FILE* fp) { if (fp) fclose(fp); };
std::unique_ptr<FILE, decltype(deleter)> file(fopen("example.txt", "r"), deleter);
```

std::inout_ptr 在这里没有用武之地，因为 fopen 不符合 std::inout_ptr 的使用模式。

------

2. 构造一个适合 std::inout_ptr 的场景

为了展示 std::inout_ptr 的用法，我们假设有一个函数 reopen_file，它的签名是：

cpp

```cpp
int reopen_file(const char* filename, const char* mode, FILE** in_out);
```

这个函数：

- 接受一个 FILE** in_out 参数。
- 如果 *in_out 不为空，会关闭旧的文件（调用 fclose）。
- 打开一个新文件，并将新的 FILE* 写入 *in_out。
- 返回 0 表示成功，-1 表示失败。

使用 std::inout_ptr 改写

以下是完整的代码：

cpp

```cpp
#include <memory>
#include <cstdio>
#include <iostream>

// 假设的函数：重新打开文件
int reopen_file(const char* filename, const char* mode, FILE** in_out) {
    // 如果 *in_out 不为空，关闭旧文件
    if (*in_out) {
        fclose(*in_out);
        *in_out = nullptr;
    }
    // 打开新文件
    *in_out = fopen(filename, mode);
    return (*in_out != nullptr) ? 0 : -1;
}

int main() {
    // 使用 lambda 作为删除器
    auto deleter = [](FILE* fp) { if (fp) fclose(fp); };
    std::unique_ptr<FILE, decltype(deleter)> file(nullptr, deleter);

    // 使用 std::inout_ptr 调用 reopen_file
    if (reopen_file("example.txt", "r", std::inout_ptr(file)) == 0) {
        char buffer[256];
        while (fgets(buffer, sizeof(buffer), file.get())) {
            printf("%s", buffer);
        }
    } else {
        std::cout << "Failed to open file\n";
    }

    // file 超出作用域时，std::unique_ptr 自动调用 deleter
    return 0;
}
```

逐步解释

1. **reopen_file 函数**：
   - 这个函数符合 std::inout_ptr 的使用场景：它接受一个 FILE** 参数，释放旧资源（如果存在），并分配新资源。
   - 它先检查 *in_out 是否为空，如果不为空就调用 fclose 关闭旧文件。
   - 然后调用 fopen 打开新文件，将结果写入 *in_out。
2. **std::inout_ptr(file)**：
   - std::inout_ptr(file) 将 std::unique_ptr 适配为一个临时的 FILE**，可以传递给 reopen_file 的第三个参数。
   - std::inout_ptr 会在调用 reopen_file 前释放 file 当前管理的资源（通过调用删除器），然后在调用后将新的 FILE* 转移给 file 管理。
   - 具体步骤：
     - std::inout_ptr 临时接管 file 的资源，调用 file.release() 释放当前 FILE*（但不会调用删除器）。
     - 将 FILE* 的地址传递给 reopen_file。
     - reopen_file 可能会关闭旧文件并分配新文件。
     - 调用结束后，std::inout_ptr 将 *in_out 的值（新 FILE*）重新交给 file 管理。
3. **删除器**：
   - std::unique_ptr 仍然使用 lambda 删除器 deleter，确保 FILE* 在析构时被正确关闭。
4. **好处**：
   - std::inout_ptr 确保了与 C 风格 API 的安全交互，避免手动管理指针。
   - 它自动处理资源的释放和重新分配，减少出错的可能性。

------

3. 如果强行用 std::inout_ptr 改写 fopen

如果我们真的想用 std::inout_ptr 来处理 fopen，需要将 fopen 封装成一个符合 std::inout_ptr 模式的函数（类似上面 reopen_file 的做法）。但这种封装显得有些多余，因为 fopen 直接返回 FILE*，用 std::inout_ptr 会增加复杂性。

封装 fopen 的例子

cpp

```cpp
#include <memory>
#include <cstdio>
#include <iostream>

// 封装 fopen 为适合 std::inout_ptr 的形式
int wrapped_fopen(const char* filename, const char* mode, FILE** in_out) {
    if (*in_out) {
        fclose(*in_out);
        *in_out = nullptr;
    }
    *in_out = fopen(filename, mode);
    return (*in_out != nullptr) ? 0 : -1;
}

int main() {
    auto deleter = [](FILE* fp) { if (fp) fclose(fp); };
    std::unique_ptr<FILE, decltype(deleter)> file(nullptr, deleter);

    // 使用 std::inout_ptr 调用封装的函数
    if (wrapped_fopen("example.txt", "r", std::inout_ptr(file)) == 0) {
        char buffer[256];
        while (fgets(buffer, sizeof(buffer), file.get())) {
            printf("%s", buffer);
        }
    } else {
        std::cout << "Failed to open file\n";
    }

    return 0;
}
```

为什么这种改写不必要？

- fopen 直接返回 FILE*，可以直接传递给 std::unique_ptr 的构造函数。
- 封装成 wrapped_fopen 并使用 std::inout_ptr 增加了代码复杂性，但没有带来实际的好处。
- std::inout_ptr 的真正价值在于处理需要 T** 参数的 C 风格 API，而不是像 fopen 这样直接返回资源的函数。

------

总结

- **直接使用 fopen 的最佳方式**：保持原始代码，不需要 std::inout_ptr。

  cpp

  ```cpp
  auto deleter = [](FILE* fp) { if (fp) fclose(fp); };
  std::unique_ptr<FILE, decltype(deleter)> file(fopen("example.txt", "r"), deleter);
  ```

- **适合 std::inout_ptr 的场景**：当你需要与一个接受 FILE** 参数的 C 风格函数交互时（比如 reopen_file），std::inout_ptr 可以简化资源管理。

- 如果你有其他具体需求（比如处理更复杂的文件操作函数），可以进一步调整代码！