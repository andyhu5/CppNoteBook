`implicitWidth` 和 `implicitHeight` 是 **QML 布局系统中最重要的概念之一**。可以把它理解为：

> **组件告诉外界："如果没有人指定我的大小，我希望自己是这么大。"**

它们表示的是**组件的推荐（自然）尺寸**，而不是最终尺寸。

------

## 1. 与 width/height 的区别

例如：

```qml
Rectangle {
    implicitWidth: 100
    implicitHeight: 50
}
```

这里表示：

> 我希望自己的大小是 **100×50**。

但是如果父对象指定：

```qml
Rectangle {
    width: 300
    height: 200
}
```

那么最终大小就是：

```text
width  = 300
height = 200
```

而不是：

```text
100 × 50
```

也就是说：

- `implicitWidth`：建议宽度
- `implicitHeight`：建议高度
- `width`：最终宽度
- `height`：最终高度

------

## 2. 为什么需要 implicitSize？

很多组件的大小是可以自己计算出来的。

例如：

```qml
Text {
    text: "Hello Qt"
}
```

你没有写：

```qml
width: ?
height: ?
```

为什么还能显示？

因为 `Text` 内部会根据：

- 字体
- 字号
- 文本长度

计算：

```text
implicitWidth  = 58
implicitHeight = 24
```

然后：

```text
width  = implicitWidth
height = implicitHeight
```

所以能正常显示。

例如：

```qml
Text {
    text: "Hello"
}

Component.onCompleted: {
    console.log(width)
    console.log(implicitWidth)
}
```

可能输出：

```text
35
35
```

------

## 3. Layout 为什么依赖 implicitSize？

例如：

```qml
ColumnLayout {

    Button {
        text: "OK"
    }

    Button {
        text: "Cancel"
    }
}
```

你没有写任何高度。

为什么按钮有高度？

因为 Button 内部定义了：

```text
implicitHeight = 40
implicitWidth = 80
```

`ColumnLayout` 就会读取：

```text
Button1：40
Button2：40
```

然后计算：

```text
整个 ColumnLayout 高度 = 40 + spacing + 40
```

------

## 4. 自定义组件为什么要设置 implicitHeight？

例如：

```qml
Rectangle {
    color: "tomato"
}
```

默认：

```text
implicitWidth = 0
implicitHeight = 0
```

放进 Layout：

```qml
ColumnLayout {

    Rectangle {
        color: "red"
    }
}
```

它可能就是：

```text
0 × 0
```

看不到。

正确写法：

```qml
Rectangle {
    implicitWidth: 200
    implicitHeight: 60
}
```

Layout 就知道：

```text
希望它是 200×60
```

------

## 5. implicit 与 Layout.preferredHeight 的区别

很多人容易混淆。

例如：

```qml
Rectangle {
    implicitHeight: 80
}
```

表示：

> 我自己的自然高度是 80。

而：

```qml
Rectangle {
    Layout.preferredHeight: 80
}
```

表示：

> 如果我是 Layout 的子项，希望 Layout 分配给我 80。

两者层次不同：

```text
implicitHeight
    ↑
组件自身告诉别人

Layout.preferredHeight
    ↑
布局属性，告诉 Layout 如何分配空间
```

------

## 6. 为什么你的 ListView 高度是 0？

你之前遇到：

```text
ListView
height = 0
```

就是因为：

`ListView` 的

```text
implicitHeight = 0
```

而你又没有：

```qml
Layout.preferredHeight
```

于是 Layout 无法推导出它应该有多高。

像 `ListView`、`Flickable`、`ScrollView` 这类可滚动控件，它们通常**没有有意义的隐式高度**，因为它们可以显示无限多的数据，框架无法猜测它们应该有多高。因此，它们经常需要通过 `Layout.fillHeight`（由父布局分配空间）或显式高度来确定尺寸。

------

## 7. 查看 implicitSize

可以打印：

```qml
Rectangle {
    width: 300
    height: 100

    Component.onCompleted: {
        console.log(width, height)
        console.log(implicitWidth, implicitHeight)
    }
}
```

例如：

```text
300 100
0 0
```

因为 `Rectangle` 默认没有隐式大小。

而：

```qml
Button {
    text: "Hello"

    Component.onCompleted: {
        console.log(width, implicitWidth)
        console.log(height, implicitHeight)
    }
}
```

可能输出：

```text
86 86
40 40
```

因为 `Button` 自己会计算推荐尺寸。

------

### 一句话总结

可以把它们理解成下面四个概念：

| 属性             | 含义                           |
| ---------------- | ------------------------------ |
| `implicitWidth`  | 组件**希望**有多宽（自然宽度） |
| `implicitHeight` | 组件**希望**有多高（自然高度） |
| `width`          | 实际宽度                       |
| `height`         | 实际高度                       |

在 QML 中，`Layout`、`Control`、`Text`、`Image` 等组件大量依赖 `implicitWidth` 和 `implicitHeight` 来进行自动布局，因此理解它们是掌握 QML 布局机制的关键。