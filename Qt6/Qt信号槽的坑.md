是的，你现在已经抓住了这个问题的本质。

准确地说，**不是因为信号需要 QML 监听，而是因为你操作的不是同一个 `Backend` 实例。**

如果使用 `QML_SINGLETON`，Qt 的生命周期大致如下：

```text
           QML Engine
                │
        创建 Backend 单例
                │
        Backend::Backend()
                │
        Button.onClicked
                │
        Backend.Start()
```

也就是说，**QML 调用的 `Start()` 一定属于 Qt 创建的那个单例对象**。

如果你在 `main()` 又写了：

```cpp
Backend backend;

connect(&backend,
        &Backend::Done,
        &backend,
        &Backend::HandleDone);
```

那么实际上是：

```text
Backend A（main 创建）
    │
    ├── connect(Done, HandleDone)
    └── 从未调用 Start()

Backend B（QML_SINGLETON 创建）
    │
    ├── Start()
    └── emit Done()
```

这两个对象互不相干，因此 `emit Done()` 不会触发 `Backend A` 上建立的连接。

------

### 正确的做法有两种

**方法一（推荐）**：在构造函数中建立连接。

```cpp
Backend::Backend(QObject *parent)
    : QObject(parent)
{
    connect(this,
            &Backend::Done,
            this,
            &Backend::HandleDone);
}
```

这样无论是谁创建 `Backend`，每个实例都会自动建立自己的连接。

------

**方法二**：如果确实需要在外部 `connect()`，那么必须拿到 **QML 使用的那个实例**，而不是自己 `new` 一个或在栈上创建一个新的 `Backend`。

------

### 这是 Qt 初学者非常容易踩的坑

很多人会误以为：

```text
Backend
```

表示的是"这个类"。

实际上在 Qt/QML 中，所有的 signal/slot 都是**对象（QObject 实例）级别**的，而不是类级别的。

因此：

```text
Backend 类
    │
    ├── Backend 对象 A
    │      ├── Done
    │      └── HandleDone
    │
    └── Backend 对象 B
           ├── Done
           └── HandleDone
```

A 发出的 `Done()`，只会通知连接到 **A** 的槽函数，不会通知 B。

------

这是学习 Qt 信号槽时一个非常重要的概念：

> **`QObject::connect()` 连接的是两个具体的对象（实例），而不是两个类。**

理解了这一点，后面学习 `QML_SINGLETON`、`qmlRegisterSingletonType()`、`Connections`、跨线程信号槽等内容都会清晰很多。