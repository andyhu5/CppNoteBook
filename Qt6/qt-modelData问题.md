问题出在这一行：

```qml
text: modelData.name
```

**`ListModel` + `Repeater`** 与 JavaScript 数组不同，delegate 中并没有 `modelData` 对象。

对于 `ListModel`，每个 `ListElement` 的角色（role）会直接变成 delegate 的属性，因此应该直接使用 `name` 和 `count`。

修改如下：

```qml
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts

ApplicationWindow {
    id: window
    width: 640
    height: 480
    visible: true

    ListModel {
        id: tags

        ListElement {
            name: "Apple"
            count: 100
        }

        ListElement {
            name: "BlackBerry"
            count: 200
        }

        ListElement {
            name: "Peak"
            count: 300
        }
    }

    Flow {
        anchors.fill: parent
        spacing: 10

        Repeater {
            model: tags

            delegate: Button {
                text: name
                onClicked: {
                    console.log(name, count)
                }
            }
        }
    }
}
```

输出：

```
Apple 100
BlackBerry 200
Peak 300
```

------

### 为什么 `modelData` 不行？

`Repeater` 的 `model` 可以有多种类型，不同模型暴露的数据不同。

| model 类型                 | delegate 中访问方式              |
| -------------------------- | -------------------------------- |
| `ListModel`                | `name`、`count`（直接使用 role） |
| JavaScript Array（字符串） | `modelData`                      |
| JavaScript Array（对象）   | `modelData.name`                 |
| 整数（如 `model: 5`）      | `index`                          |
| C++ `QAbstractListModel`   | role 名（例如 `name`）           |

例如：

### JavaScript 数组

```qml
property var fruits: [
    {name: "Apple"},
    {name: "Banana"},
    {name: "Orange"}
]

Repeater {
    model: fruits

    delegate: Button {
        text: modelData.name   // 正确
    }
}
```

### ListModel

```qml
ListModel {
    ListElement { name: "Apple" }
}

Repeater {
    model: tags

    delegate: Button {
        text: name            // 正确
        // text: modelData.name // 错误
    }
}
```

### Qt 6 中更推荐的写法

为了避免 role 名称与局部变量冲突，Qt 6 推荐使用 `required property` 显式声明需要的角色：

```qml
Repeater {
    model: tags

    delegate: Button {
        required property string name
        required property int count

        text: name

        onClicked: {
            console.log(name, count)
        }
    }
}
```

这种写法具有几个优点：

- 编译器能够检查 role 是否存在。
- Qt Creator 能提供更好的自动补全。
- role 名称更加明确，可维护性更高。
- 这是 Qt 6 官方推荐的 delegate 编写方式。