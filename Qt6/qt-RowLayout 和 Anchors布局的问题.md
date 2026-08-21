RowLayout 和 Anchors布局的问题

**原则：一个 Item 的几何（位置、大小）只能由一种机制管理。** 如果一个 Item 已经由 `RowLayout` 管理，就不要再用 `anchors`、`x/y`、`width/height` 去控制它的位置和大小。

你的代码：

```qml
RowLayout {
    anchors.fill: parent

    Image {
        anchors.leftMargin: 0      // ❌
        Layout.preferredWidth: 50
        Layout.preferredHeight: 50
    }

    Text {
        ...
    }
}
```

这里的：

```qml
anchors.leftMargin: 0
```

**不会生效。**

原因是 `Image` 是 `RowLayout` 的直接子项，它的位置由 `RowLayout` 决定，而不是由 `anchors` 决定。

------

## 哪些可以混用？

### ① Layout 自己可以使用 anchors（✔）

这是最常见的：

```qml
Rectangle {
    RowLayout {
        anchors.fill: parent
    }
}
```

这里 `anchors` 控制的是 **RowLayout**，没有问题。

------

### ② Layout 管理的子项不要使用 anchors（✘）

例如：

```qml
RowLayout {

    Rectangle {
        anchors.left: parent.left   // ❌
    }
}
```

Qt 通常会输出类似警告：

```
Detected anchors on an item that is managed by a layout.
This is undefined behavior.
```

------

### ③ 非 Layout 子项可以使用 anchors（✔）

例如：

```qml
Rectangle {

    Image {
        anchors.left: parent.left
        anchors.verticalCenter: parent.verticalCenter
    }
}
```

这里没有 Layout，完全没问题。

------

## Layout 中应该怎么做？

如果以前写：

```qml
anchors.leftMargin: 10
```

应该改成：

```qml
Layout.leftMargin: 10
```

如果以前写：

```qml
anchors.rightMargin: 10
```

改成：

```qml
Layout.rightMargin: 10
```

如果以前写：

```qml
anchors.verticalCenter: parent.verticalCenter
```

改成：

```qml
Layout.alignment: Qt.AlignVCenter
```

如果以前写：

```qml
anchors.horizontalCenter: parent.horizontalCenter
```

改成：

```qml
Layout.alignment: Qt.AlignHCenter
```

------

## 一个对照表

| 需求     | 普通 `Item`              | `RowLayout` / `ColumnLayout`                                 |
| -------- | ------------------------ | ------------------------------------------------------------ |
| 左对齐   | `anchors.left`           | 自动按顺序排列                                               |
| 右对齐   | `anchors.right`          | `Layout.fillWidth` + `horizontalAlignment` 或 `Layout.alignment` |
| 垂直居中 | `anchors.verticalCenter` | `Layout.alignment: Qt.AlignVCenter`                          |
| 左边距   | `anchors.leftMargin`     | `Layout.leftMargin`                                          |
| 右边距   | `anchors.rightMargin`    | `Layout.rightMargin`                                         |
| 大小     | `width`、`height`        | `Layout.preferredWidth`、`Layout.preferredHeight`            |

------

### 你的代码应改为

```qml
RowLayout {
    anchors.fill: parent
    spacing: 15

    Image {
        source: "qrc:/images/WeiyunApp.png"

        Layout.preferredWidth: 50
        Layout.preferredHeight: 50
        Layout.alignment: Qt.AlignVCenter

        fillMode: Image.PreserveAspectFit
    }

    Text {
        text: "Hello world"
        color: "white"

        font.family: "Consolas"
        Layout.alignment: Qt.AlignVCenter
    }
}
```

这符合 Qt Quick Layout 的设计方式，不会出现 `anchors` 与布局管理冲突的问题。