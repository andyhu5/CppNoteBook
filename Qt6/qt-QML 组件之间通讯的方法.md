QML 组件之间通讯的方法主要有以下几种，按使用场景和复杂度分类：

## 1. 信号与槽（Signal & Slot）— 最常用

**子组件发送信号，父组件监听：**

```qml
// Child.qml
Item {
    signal buttonClicked(string message)
    
    MouseArea {
        anchors.fill: parent
        onClicked: buttonClicked("Hello from child")
    }
}

// Parent.qml
Child {
    onButtonClicked: (message) => {
        console.log("收到:", message)
    }
}
```

**适用场景**：子向父通信，事件通知（如点击、状态改变）。

## 2. 属性绑定（Property Binding）

```qml
// 直接绑定属性，自动同步
Rectangle {
    id: rect
    property int count: 0
}

Text {
    text: rect.count  // 自动响应 count 变化
}
```

**双向绑定**（Qt6 推荐用法）：

```qml
Slider {
    id: slider
    value: 50
}

TextField {
    text: slider.value
    onTextChanged: slider.value = parseInt(text)
}
```

或使用 `Binding` 元素做双向同步。

**适用场景**：数据同步、UI 联动。

## 3. 直接访问 id（父访问子 / 兄弟组件）

```qml
Item {
    Rectangle {
        id: rect1
        color: "red"
    }
    Rectangle {
        id: rect2
        width: 100
        color: rect1.color  // 直接访问兄弟组件属性
    }
}
```

**⚠️ 限制**：只能在同一 QML 文件作用域内使用，跨文件无法直接用 id 访问。

## 4. 函数调用（暴露方法）

```qml
// Child.qml
Item {
    function doSomething(param) {
        console.log("执行:", param)
    }
}

// Parent.qml
Child {
    id: child
    Component.onCompleted: child.doSomething("test")
}
```

**适用场景**：父调用子的方法（命令式操作）。

## 5. 单例（Singleton）— 全局状态管理

```qml
// AppState.qml
pragma Singleton
import QtQuick

QtObject {
    property int globalCounter: 0
    signal dataChanged(var data)
}
```

在 `qmldir` 中注册：

```
singleton AppState 1.0 AppState.qml
```

使用：

```qml
import "." 

Text {
    text: AppState.globalCounter
}

Button {
    onClicked: AppState.globalCounter++
}
```

**适用场景**：跨多个不相关组件共享全局状态，无需层层传递。

## 6. Context Property（C++ 注入）

在 C++ 端注册对象供整个 QML 树访问：

```cpp
QQmlApplicationEngine engine;
MyDataModel model;
engine.rootContext()->setContextProperty("dataModel", &model);
Text {
    text: dataModel.someProperty
}
```

**适用场景**：C++ 后端与 QML 前端通信，全局共享的业务逻辑对象。

## 7. Connections 元素（动态连接）

```qml
Connections {
    target: someObject
    function onSignalName(param) {
        console.log("信号触发:", param)
    }
}
```

**适用场景**：需要动态切换监听目标，或在非直接父子关系间监听信号。

## 8. QML 与 C++ 交互（Q_INVOKABLE / Q_PROPERTY）

```cpp
class Backend : public QObject {
    Q_OBJECT
    Q_PROPERTY(QString status READ status NOTIFY statusChanged)
public:
    Q_INVOKABLE void performAction(const QString &param);
signals:
    void statusChanged();
};
Backend {
    id: backend
}

Button {
    onClicked: backend.performAction("test")
}

Text {
    text: backend.status
}
```

**适用场景**：QML 调用 C++ 逻辑，C++ 通知 QML 更新。

## 9. EventBus 模式（自定义全局信号总线）

结合单例 + 信号，实现类似发布订阅的机制：

```qml
// EventBus.qml (Singleton)
pragma Singleton
QtObject {
    signal messageSent(string channel, var data)
}
```

任意组件都可以：

```qml
EventBus.messageSent.connect(function(channel, data) {...})
EventBus.messageSent("chat", {text: "hello"})
```

**适用场景**：复杂应用中解耦多个不相关模块的通信。

------

## 总结对比表

| 方法             | 通信方向  | 适用场景       | 复杂度 |
| ---------------- | --------- | -------------- | ------ |
| 信号/槽          | 子→父     | 事件通知       | 低     |
| 属性绑定         | 双向/单向 | 数据同步       | 低     |
| id 直接访问      | 双向      | 同文件内组件   | 低     |
| 函数调用         | 父→子     | 命令式操作     | 低     |
| Singleton        | 全局      | 跨组件共享状态 | 中     |
| Context Property | C++→QML   | 后端注入       | 中     |
| Connections      | 动态监听  | 动态目标切换   | 中     |
| C++ 交互         | 双向      | 业务逻辑处理   | 高     |
| EventBus         | 全局      | 解耦复杂通信   | 中高   |

**实际项目建议**：

- **父子组件**：信号 + 属性绑定
- **兄弟组件**：通过共同父组件中转，或用 id
- **跨模块/全局状态**：Singleton 或 C++ Context Property
- **业务逻辑复杂**：交给 C++ 处理，QML 只做展示层

如果你有具体的通信场景（比如 Andy 和 Tom 之间要传递什么数据），我可以给出更针对性的实现代码。