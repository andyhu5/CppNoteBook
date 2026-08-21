## Loader的用法

```json
import QtQuick
import QtQuick.Layouts
import QtQuick.Controls.Basic
import "."

ApplicationWindow {
    id: window
    width: 640
    height: 480
    minimumWidth: 200
    minimumHeight: 250
    visible: true
    title: qsTr("Hello World")

    Component {
        id: redSquare
        RedSquare {
            myColor: "green"
        }
    }

    Flow {
        spacing: 10
        Repeater {
            model: 100
            Loader {
                // source: "RedSquare.qml"
                sourceComponent: redSquare
            }
        }
    }
}
```



```json
//RedSquare.qml

import QtQuick
import QtQuick.Layouts
import QtQuick.Controls.Basic

Rectangle {
    property color myColor: "#FFFF0000"

    id: redSquare
    color: myColor
    width: 10
    height: 10
}

```



## 方法一：使用 source（推荐）

如果 `RedSquare.qml` 与当前 QML 文件在同一个目录：

```qml
Flow {
    spacing: 10

    Repeater {
        model: 100

        Loader {
            source: "RedSquare.qml"
        }
    }
}
```

运行后会动态创建 100 个 `RedSquare`。

------

## 方法二：使用 sourceComponent

如果已经有一个 `Component`：

```qml
Component {
    id: redSquareComponent

    RedSquare {
    }
}
```

然后：

```qml
Repeater {
    model: 100

    Loader {
        sourceComponent: redSquareComponent
    }
}
```

这里 `sourceComponent` 指向的是 `Component`。

------

## 方法三：直接创建（最快）

实际上你的例子完全没必要使用 Loader。

```qml
Flow {
    spacing: 10

    Repeater {
        model: 100

        RedSquare {
        }
    }
}
```

这也是官方推荐的方式。

------

# Loader 的真正用途

很多初学者认为 Loader 就是"创建组件"，其实它更重要的作用是**按需加载和卸载**。

例如：

```qml
Loader {
    id: loader
    active: false
}
```

点击按钮：

```qml
Button {
    text: "Load"

    onClicked: {
        loader.source = "RedSquare.qml"
        loader.active = true
    }
}
```

关闭：

```qml
loader.active = false
```

这时：

- 创建对象
- 销毁对象
- 释放内存

都是 Loader 自动完成的。

------

# Loader 和 Repeater 的区别

| 功能              | Loader                            | Repeater      |
| ----------------- | --------------------------------- | ------------- |
| 创建一个组件      | ✔                                 | ✘（创建多个） |
| 动态加载 qml 文件 | ✔                                 | ✘             |
| 延迟加载          | ✔                                 | ✘             |
| 卸载释放对象      | ✔ (`active=false` 或 `source=""`) | ✘             |
| 大量重复创建      | ✘                                 | ✔             |

因此，对于你的例子：

```qml
Repeater {
    model: 100
    RedSquare { }
}
```

比

```qml
Repeater {
    model: 100
    Loader {
        source: "RedSquare.qml"
    }
}
```

效率更高，因为 `Loader` 本身也是一个 QML 对象，每个 `Loader` 都会增加一层对象和管理开销。如果只是为了重复创建 `RedSquare`，直接使用 `RedSquare {}` 更简单也更高效；只有在需要**延迟加载、按条件创建或运行时切换组件**时，才建议使用 `Loader`。