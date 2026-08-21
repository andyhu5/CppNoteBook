

QML 中与初始化相关的机制主要有下面几种。

## 1. `Component.onCompleted`（最常用）

几乎所有继承自 `QObject` 的 QML 对象都可以使用 **`Component.onCompleted`**。

```qml
Rectangle {
    Component.onCompleted: {
        console.log("Rectangle initialized")
    }
}
```

它表示：

> **当前对象已经创建完成，所有属性已初始化，并加入对象树。**

这是 QML 中最常用的初始化入口。

------

## 2. `Component.onDestruction`

对应销毁时：

```qml
Rectangle {
    Component.onDestruction: {
        console.log("Rectangle destroyed")
    }
}
```

------

## 3. 属性变化处理

如果初始化依赖某个属性，可以监听：

```qml
Text {
    onTextChanged: {
        console.log(text)
    }
}
```

或者：

```qml
Switch {
    onCheckedChanged: {
        console.log(checked)
    }
}
```

------

## 4. 特定组件自己的信号

有些组件提供自己的生命周期信号，例如：

```qml
Loader {
    onLoaded: {
        console.log("Component loaded")
    }
}
```

这里的 `onLoaded` **仅属于 `Loader`**，不是所有组件都有。

再例如：

```qml
Image {
    onStatusChanged: {
        if (status === Image.Ready)
            console.log("Image loaded")
    }
}
```

也是 `Image` 特有的。

------

## `Component.onCompleted` 的执行时机

例如：

```qml
ApplicationWindow {
    Component.onCompleted: console.log("Window")

    Rectangle {
        Component.onCompleted: console.log("Rectangle")
    }
}
```

通常会在对象创建完成后分别执行各自的 `Component.onCompleted`。需要注意的是，对于父子对象，不应依赖它们之间严格的执行先后顺序；如果存在初始化依赖，应通过属性绑定、信号或显式调用来协调，而不是假设某个 `Component.onCompleted` 一定先执行。

------

## C++ 对应关系

如果熟悉 Qt Widgets 或 C++，可以类比：

| C++      | QML                       |
| -------- | ------------------------- |
| 构造函数 | 属性初始化                |
| 构造结束 | `Component.onCompleted`   |
| 析构函数 | `Component.onDestruction` |

因此：

```qml
Component.onCompleted
```

可以近似理解为：

```cpp
MyWidget::MyWidget()
{
    // 构造完成后的初始化
}
```

------

