# C++23：`std::unreachable()` 解释

`std::unreachable()` 是 C++23 在 `<utility>` 头文件中新增的一个非常简单但很实用的函数，来自提案 P0627。它的作用是**告诉编译器"这段代码逻辑上永远不会被执行到"**，从而让编译器可以据此做更激进的优化，同时也起到文档化代码意图的作用。

## 1. 函数签名

```cpp
namespace std {
    [[noreturn]] void unreachable();
}
```

- 标记为 `[[noreturn]]`：表示它不会正常返回
- **调用它是未定义行为（UB）**——如果程序运行时真的执行到了这一行，那就是 UB，而不是抛异常或者 abort

## 2. 典型用法：标记"不可能到达"的分支

最常见的场景是 `switch` 语句处理完所有已知情况后的 `default` 分支，或者其他逻辑上已经排除的代码路径：

```cpp
#include <utility>

enum class Color { Red, Green, Blue };

int to_code(Color c) {
    switch (c) {
        case Color::Red:   return 1;
        case Color::Green: return 2;
        case Color::Blue:  return 3;
    }
    std::unreachable();  // 逻辑上不会执行到这里（除非枚举被非法构造）
}
```

对比 C++23 之前常见的写法：

```cpp
// 之前只能这样"骗"编译器，或者依赖编译器扩展
int to_code(Color c) {
    switch (c) {
        case Color::Red:   return 1;
        case Color::Green: return 2;
        case Color::Blue:  return 3;
    }
    assert(false && "unreachable");   // release 模式下 assert 被禁用，起不到优化作用
    return -1;                         // 或者硬写一个"占位"返回值，语义上很别扭
}

// 各编译器私有扩展（不可移植）
__builtin_unreachable();        // GCC/Clang
__assume(0);                    // MSVC
```

`std::unreachable()` 就是把这些编译器私有扩展**标准化**了，一份代码，各编译器都能识别。

## 3. 和 `assert(false)` 的本质区别

这是最容易搞混的一点：

|                          | `assert(false)`                | `std::unreachable()`                   |
| ------------------------ | ------------------------------ | -------------------------------------- |
| Debug 模式               | 触发断言失败，程序中止         | 仍然是 UB（不保证任何具体行为）        |
| Release 模式（`NDEBUG`） | 直接被预处理器移除，什么都不做 | 依然向编译器传达"不可达"信息，用于优化 |
| 目的                     | 运行时检查 + 调试辅助          | 纯粹的优化提示 + 文档化意图            |

也就是说 `std::unreachable()` **不是用来做安全检查的**，它不会帮你在运行时"兜底"报错。如果你的假设错了、真的执行到这里，那就是纯粹的 UB（可能崩溃、可能输出垃圾、可能什么问题都不表现出来直到很久以后）。

如果你既想要 debug 期间的安全检查，又想要 release 期间的优化提示，通常的做法是两者结合：

```cpp
int to_code(Color c) {
    switch (c) {
        case Color::Red:   return 1;
        case Color::Green: return 2;
        case Color::Blue:  return 3;
    }
    assert(false && "unexpected Color value");  // debug: 帮你抓 bug
    std::unreachable();                          // release: 帮编译器优化
}
```

## 4. 对优化的实际影响

编译器知道某分支不可达之后，可以：

- **省略生成该分支对应的代码**，减小二进制体积
- **省略对该分支的边界检查/跳转表兜底项**，让 `switch` 生成的跳转表更紧凑
- 帮助**其他分析**（比如认为函数在某分支之后变量一定有效，从而消除多余检查）

例如上面 `to_code` 的例子，编译器知道 `switch` 已经穷尽所有 `enum class Color` 的合法值（前提是没人用 `static_cast` 硬塞一个非法值进去），加了 `std::unreachable()` 之后，编译器甚至可能**完全不为 default 分支生成任何返回值处理代码**，比手写一个 `return -1;` 更紧凑、也更清楚地表达"这不是一个真实的业务分支"。

## 5. 使用时的注意事项

- **前提假设必须真的成立**，否则是货真价实的 UB，比普通逻辑 bug 更危险（因为编译器可能基于"不可达"做出激进优化，一旦假设错误，行为可能比"什么都不做"更离谱，比如整个函数被优化到面目全非）
- 适合用在你**非常确信**（通过类型系统、前置校验、枚举穷尽性等）不会发生的分支，而不是"图省事随手放一个"
- 常见搭配场景：穷尽性 `switch`、状态机中"不可能的状态转换"、模板元编程中"这个分支在当前实例化下必然不成立"等

## 6. 一句话总结

**`std::unreachable()` 是标准化的"告诉编译器这里绝对到不了"的提示函数，本质是优化提示而非运行时检查，用对了能帮编译器生成更紧凑高效的代码，但如果假设错误，调用它就是纯粹的未定义行为。**