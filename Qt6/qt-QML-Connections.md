`Connections` 是 QML 中专门用来**连接（监听）QObject 信号（signal）**的组件。

它的作用相当于 C++ 中的：

```cpp
QObject::connect(sender,
                 &Sender::signal,
                 receiver,
                 &Receiver::slot);
```

可以理解为：

> **Connections = QML 中的信号槽（Signal/Slot）连接器。**

------

# 为什么需要 Connections？

假设有一个按钮：

```qml
Button {
    onClicked: {
        console.log("Clicked")
    }
}
```

这里为什么不需要 `Connections`？

因为：

```qml
onClicked
```

实际上就是 Button 自己提供的 signal handler。

它等价于：

```qml
Connections {
    target: button

    function onClicked() {
        console.log("Clicked")
    }
}
```

因此：

**对于当前对象自己的 signal，一般不用 Connections。**

------

# Connections 最大用途

它可以监听**其他对象**的 signal。

例如：

```qml
ApplicationWindow {
    id: window

    Button {
        id: btn
        text: "Click"
    }

    Connections {
        target: btn

        function onClicked() {
            console.log("Button Clicked")
        }
    }
}
```

这里：

```text
Button
   │
clicked()
   │
   ▼
Connections
```

按钮并没有写：

```qml
onClicked:
```

而是在外部监听。

------

# 一个更实际的例子

例如 Backend：

```cpp
class Backend : public QObject
{
    Q_OBJECT

signals:
    void loginSuccess(QString user);
};
```

注册到 QML：

```cpp
engine.rootContext()->setContextProperty("Backend",&backend);
```

QML：

```qml
Connections {
    target: Backend

    function onLoginSuccess(user) {
        console.log(user)
    }
}
```

当 C++：

```cpp
emit loginSuccess("Andy");
```

QML：

```text
Andy
```

立即打印。

------

# 监听多个对象

例如：

```qml
Button {
    id: saveBtn
}

Button {
    id: openBtn
}
```

可以：

```qml
Connections {
    target: saveBtn

    function onClicked() {
        console.log("Save")
    }
}

Connections {
    target: openBtn

    function onClicked() {
        console.log("Open")
    }
}
```

不会影响 Button 自己。

------

# target 可以改变

例如：

```qml
property var currentObject
Connections {
    target: currentObject

    function onValueChanged(v) {
        console.log(v)
    }
}
```

如果：

```qml
currentObject = sensor1
```

监听：

```text
sensor1
```

后来：

```qml
currentObject = sensor2
```

自动改为监听：

```text
sensor2
```

不用重新 connect。

这是 C++ 很难做到的。

------

# enabled

Connections 可以关闭。

```qml
Connections {
    target: Backend

    enabled: false

    function onDataReady() {

    }
}
```

此时：

```cpp
emit dataReady();
```

QML 不会收到。

------

# ignoreUnknownSignals

例如：

```qml
Connections {
    target: obj

    ignoreUnknownSignals: true
}
```

如果：

```qml
function onABC() {}
```

而：

```text
obj
```

没有：

```text
ABC
```

不会报错。

对于动态对象非常有用。

------

# 与直接写 onSignal 的区别

例如：

```qml
Button {

    onClicked: {

    }
}
```

优点：

- 简单
- 推荐
- 当前对象

而：

```qml
Connections {
    target: button

    function onClicked() {

    }
}
```

优点：

- 可以监听任何 QObject
- 一个对象可有多个监听者
- target 可以动态改变
- 不修改原对象

------

# Qt/QML 项目中最常见的用途

几乎所有 Qt Quick 项目都会这样写：

```qml
Connections {
    target: Backend

    function onMessageReceived(msg) {
        console.log(msg)
    }

    function onProgressChanged(value) {
        progressBar.value = value
    }

    function onErrorOccurred(err) {
        errorDialog.msg = err
        errorDialog.open()
    }
}
```

Backend：

```cpp
emit messageReceived(...);
emit progressChanged(...);
emit errorOccurred(...);
```

QML 自动响应。

------

# 总结

`Connections` 最适合下面几种场景：

1. **监听 C++ 后端对象的信号**（最常见）。
2. **监听其他 QML 对象的信号**，无需修改该对象。
3. **动态切换监听目标**，通过修改 `target` 自动重新建立连接。
4. **实现多个监听者**，同一个信号可以由多个 `Connections` 同时响应。

而对于对象自身的信号（如 `Button` 的 `clicked`），通常直接使用 `onClicked` 即可，不必额外使用 `Connections`。