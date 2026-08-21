这是 QML Layout 系统的核心概念。

`RowLayout`、`ColumnLayout`、`GridLayout` 中所有以 **`Layout.`** 开头的属性，并不是 `Item` 自身的属性，而是 **Layout Attached Properties（附加属性）**，它们用于**告诉父布局如何安排当前 Item**。

例如：

```q
RowLayout {
    anchors.fill: parent

    Rectangle {
        color: "red"

        Layout.preferredWidth: 100
        Layout.preferredHeight: 50
    }

    Rectangle {
        color: "blue"

        Layout.fillWidth: true
        Layout.preferredHeight: 50
    }
}
```

这里：

- `Rectangle` 不知道什么是 `Layout.preferredWidth`
- `RowLayout` 会读取这些 `Layout.*` 属性，并据此计算每个子项的大小和位置。

因此，可以理解为：

```
Rectangle
    │
    ├── width
    ├── height
    └── Layout.xxx  ← 给父 Layout 的"建议"
                    ↓
             RowLayout 负责读取并计算布局
```

------

## 为什么不用 `width`、`height`？

因为布局管理器希望自己控制几何信息。

例如：

```qml
Rectangle {
    width: 100
}
```

在普通 `Item` 中，`width` 会生效。

但放到 `RowLayout` 中：

```qml
RowLayout {
    Rectangle {
        width: 100
    }
}
```

通常 `width` 会被布局重新计算，甚至可能出现类似警告：

```
Detected anchors on an item that is managed by a layout.
This is undefined behavior.
```

布局认为：

> **"位置和大小由我决定，不应该由子 Item 自己决定。"**

因此，应使用：

```qml
Layout.preferredWidth: 100
```

而不是：

```qml
width: 100
```

------

## 常用的 `Layout.*` 属性

### `Layout.preferredWidth`

希望的宽度。

```qml
Layout.preferredWidth: 150
```

类似 HTML/CSS 中的 `preferred size`。

------

### `Layout.preferredHeight`

希望的高度。

```qml
Layout.preferredHeight: 80
```

------

### `Layout.minimumWidth`

最小宽度。

```qml
Layout.minimumWidth: 60
```

布局不会把它缩得比这个更小。

------

### `Layout.maximumWidth`

最大宽度。

```qml
Layout.maximumWidth: 300
```

布局不会把它拉得比这个更大。

------

### `Layout.fillWidth`

是否占据剩余空间。

```qml
Layout.fillWidth: true
```

例如：

```qml
RowLayout {

    Button {
        text: "OK"
    }

    TextField {
        Layout.fillWidth: true
    }
}
```

结果：

```
+--------------------------------+
| OK | TextField................ |
+--------------------------------+
```

按钮保持固定大小，而输入框自动填满剩余空间。

------

### `Layout.fillHeight`

同理，填满垂直方向。

------

### `Layout.alignment`

控制子项在分配区域中的对齐方式。

```qml
Layout.alignment: Qt.AlignCenter
```

也可以组合：

```qml
Layout.alignment: Qt.AlignLeft | Qt.AlignTop
```

------

### `Layout.row` / `Layout.column`

仅适用于 `GridLayout`。

```qml
Rectangle {
    Layout.row: 1
    Layout.column: 2
}
```

表示放在第 1 行、第 2 列。

------

## 为什么叫 Attached Property？

因为它不是 `Rectangle` 的成员，而是布局系统附加给子项的一组元数据。

例如：

```qml
Rectangle {
    Layout.fillWidth: true
}
```

实际上等价于：

```
Rectangle
   +
LayoutAttached
```

可以理解为：

```
Rectangle
 ├── x
 ├── y
 ├── width
 ├── height
 └── Layout
      ├── fillWidth
      ├── preferredWidth
      ├── alignment
      └── ...
```

这种设计在 QML 中很常见。除了 `Layout`，还有很多类似的附加属性，例如：

- `Layout.*`：供 `RowLayout`、`ColumnLayout`、`GridLayout` 使用。
- `GridView.*`：供 `GridView` 的 delegate 使用。
- `ListView.*`：供 `ListView` 的 delegate 使用。
- `StackLayout.*`：供 `StackLayout` 使用。

这些属性都不是子项自身的属性，而是由特定的父对象或框架读取并解释。

总结来说，`Layout.*` 的语义可以概括为：**子项向父布局声明自己的布局需求（尺寸、对齐、伸缩能力等），而父布局根据所有子项的需求统一计算最终的位置和大小。**这也是 QML 布局系统区别于直接设置 `x`、`y`、`width`、`height` 的核心思想。