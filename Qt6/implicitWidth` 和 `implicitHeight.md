有，而且**绝大多数 Qt Quick Controls 控件都有自己的 `implicitWidth` 和 `implicitHeight`**。可以把它们分成两大类来看。

## 第一类：Control（控件）——几乎都有默认尺寸

这些控件为了保证可用性，都会计算自己的推荐大小。

例如：

```qml
Button
CheckBox
RadioButton
Switch
Slider
SpinBox
ComboBox
TextField
TextArea
Label
MenuBar
ToolBar
ProgressBar
BusyIndicator
Page
TabBar
TabButton
```

例如：

```qml
Button {
    text: "OK"
}
```

内部会根据：

- 文本长度
- 字体
- padding
- background

自动计算：

```text
implicitWidth
implicitHeight
```

所以不用指定大小也能正常显示。

------

## 第二类：Qt Quick 基础 Item

这类组件很多**默认没有隐式尺寸**。

例如：

```qml
Item
Rectangle
MouseArea
FocusScope
```

默认都是：

```text
implicitWidth = 0
implicitHeight = 0
```

例如：

```qml
Rectangle {
    color: "red"
}
```

打印：

```qml
Component.onCompleted: {
    console.log(implicitWidth)
    console.log(implicitHeight)
}
```

输出通常：

```text
0
0
```

因此：

```qml
ColumnLayout {
    Rectangle {
        color: "red"
    }
}
```

什么都看不到。

必须：

```qml
Rectangle {
    implicitWidth: 200
    implicitHeight: 50
}
```

或者

```qml
Rectangle {
    width: 200
    height: 50
}
```

------

## Text、Image 属于特殊情况

它们不是 Controls，但是会计算自己的大小。

### Text

```qml
Text {
    text: "Hello"
}
```

自动计算：

```text
implicitWidth
implicitHeight
```

依据：

- 字体
- 字号
- 文本内容

------

### Image

```qml
Image {
    source: "cat.png"
}
```

加载完成后：

```text
implicitWidth = 图片宽度
implicitHeight = 图片高度
```

例如图片：

```text
800 × 600
```

那么：

```text
implicitWidth = 800
implicitHeight = 600
```

------

## ListView、GridView、Flickable

这一类比较特殊。

例如：

```qml
ListView {
}
```

默认：

```text
implicitWidth = 0
implicitHeight = 0
```

为什么？

因为：

```text
ListView
```

可以显示：

- 10 条数据
- 100 条数据
- 10000 条数据

Qt 无法知道你希望它显示多少内容。

所以：

```qml
ListView {
    Layout.fillHeight: true
}
```

或者：

```qml
height: 300
```

通常都是必须的。

------

## Row、Column

例如：

```qml
Row {
    Rectangle {
        width: 100
        height: 40
    }

    Rectangle {
        width: 200
        height: 40
    }
}
```

`Row` 会自动计算：

```text
implicitWidth = 300 + spacing
implicitHeight = 40
```

因为它知道子项大小。

`Column` 同理。

------

## RowLayout、ColumnLayout

Layout 也会计算：

```text
implicitWidth
implicitHeight
```

依据：

- 子项 implicitSize
- spacing
- margins
- Layout.preferredWidth

等等。

------

## 自定义组件

例如：

```qml
// MyButton.qml

Rectangle {
    color: "red"
}
```

默认：

```text
implicitWidth = 0
implicitHeight = 0
```

推荐：

```qml
Rectangle {
    implicitWidth: 120
    implicitHeight: 40

    color: "red"
}
```

以后：

```qml
MyButton {
}
```

Layout 就知道：

```text
希望大小：

120 × 40
```

------

## 总结

下面这张表基本涵盖了常见组件：

| 组件           | 默认 implicitSize      |
| -------------- | ---------------------- |
| `Button`       | ✔ 自动计算             |
| `TextField`    | ✔ 自动计算             |
| `CheckBox`     | ✔ 自动计算             |
| `ComboBox`     | ✔ 自动计算             |
| `Slider`       | ✔ 自动计算             |
| `Label`        | ✔ 自动计算             |
| `Text`         | ✔ 根据文字计算         |
| `Image`        | ✔ 根据图片尺寸计算     |
| `Row`          | ✔ 根据子项计算         |
| `Column`       | ✔ 根据子项计算         |
| `RowLayout`    | ✔ 根据子项计算         |
| `ColumnLayout` | ✔ 根据子项计算         |
| `Rectangle`    | ✘ 默认 0×0             |
| `Item`         | ✘ 默认 0×0             |
| `MouseArea`    | ✘ 默认 0×0             |
| `ListView`     | ✘ 默认 0×0             |
| `GridView`     | ✘ 默认 0×0             |
| `Flickable`    | ✘ 默认 0×0             |
| `Repeater`     | ✘ 非可视对象，没有尺寸 |

### 一个实用经验

可以将 QML 组件分为两类：

- **有内容（content）的组件**，通常能推导自己的尺寸，例如 `Button`、`Text`、`Image`、`CheckBox` 等。
- **容器（container）或视图（view）组件**，通常不能推导尺寸，需要由外部决定，例如 `Item`、`Rectangle`、`ListView`、`GridView`、`Flickable` 等。

理解这一点后，布局时就更容易判断什么时候需要设置 `width`、`height` 或 `Layout.fillHeight`，什么时候可以依赖组件的隐式尺寸。