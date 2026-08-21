Qt Quick（QML）的动画系统比较完整，但整体可以归纳为 **5 大类**。理解这几个分类，比记忆几十个动画类更重要。

------

# 1. Property Animation（属性动画）★★★★★

这是最核心、最常用的一类。

作用：**让某个属性在一段时间内平滑变化。**

例如：

```qml
Rectangle {
    width: 100
    height: 100
    color: "tomato"

    NumberAnimation on x {
        to: 300
        duration: 1000
    }
}
```

Qt 提供了多种属性动画：

| 动画                  | 用途                 |
| --------------------- | -------------------- |
| `PropertyAnimation`   | 通用属性动画         |
| `NumberAnimation`     | 数值（int、real）+ 1 |
| `ColorAnimation`      | 颜色 + 1             |
| `RotationAnimation`   | 旋转                 |
| `Vector3dAnimation`   | 三维向量             |
| `QuaternionAnimation` | 四元数               |

例如：

```qml
NumberAnimation {
    target: rect
    property: "opacity"
    from: 0
    to: 1
}
```

------

# 2. Behavior（行为动画）★★★★★

作用：

**当属性发生变化时，自动播放动画。**

例如：

```qml
Rectangle {
    width: 100
    height: 100

    Behavior on x {
        NumberAnimation {
            duration: 500
        }
    }

    MouseArea {
        anchors.fill: parent

        onClicked: {
            parent.x += 100
        }
    }
}
```

没有 `Behavior`：

```text
100 → 300
```

瞬间跳过去。

有 `Behavior`：

```text
100 → 120 → 160 → 210 → 300
```

平滑移动。

这是 QML 中最常见的动画方式。

------

# 3. Transition（状态切换动画）★★★★★

作用：

**两个 State 之间切换时播放动画。**

例如：

```qml
Rectangle {

    states: [
        State {
            name: "big"

            PropertyChanges {
                target: rect
                width: 300
            }
        }
    ]

    transitions: [
        Transition {
            NumberAnimation {
                properties: "width"
            }
        }
    ]
}
```

当：

```qml
state = "big"
```

时，不会立即改变，而是带动画。

Transition 专门服务于：

```text
State A
    ↓
Transition
    ↓
State B
```

------

# 4. Animator（渲染线程动画）★★★★☆

Qt 5.10 以后新增。

例如：

```qml
OpacityAnimator
ScaleAnimator
XAnimator
YAnimator
RotationAnimator
```

特点：

- 在 Render Thread 执行
- 更流畅
- 不阻塞 GUI Thread
- 适合高帧率动画

例如：

```qml
OpacityAnimator {
    target: rect
    from: 0
    to: 1
    duration: 500
}
```

一般移动端使用较多。

------

# 5. Animation Group（动画组合）★★★★★

作用：

多个动画一起播放。

Qt 提供：

### SequentialAnimation

顺序播放：

```text
Move
 ↓
Rotate
 ↓
Fade
```

例如：

```qml
SequentialAnimation {

    NumberAnimation { }

    RotationAnimation { }

    ColorAnimation { }
}
```

------

### ParallelAnimation

同时播放：

```text
Move
Rotate
Scale
```

一起开始。

例如：

```qml
ParallelAnimation {

    NumberAnimation { }

    RotationAnimation { }

    ColorAnimation { }
}
```

------

# 其它辅助动画

除了上面的五大类，还有一些特殊用途。

## SmoothedAnimation

平滑追踪目标值。

例如：

```qml
Behavior on x {
    SmoothedAnimation { }
}
```

适合：

- 鼠标跟随
- 摄像机移动

------

## SpringAnimation

弹簧效果。

例如：

```text
--------
      \
       \
        \__
```

有回弹。

用于：

- iOS 风格动画
- 卡片回弹
- 拖拽释放

------

## PauseAnimation

暂停：

```qml
SequentialAnimation {

    NumberAnimation { }

    PauseAnimation {
        duration: 500
    }

    NumberAnimation { }
}
```

------

## ScriptAction

动画过程中执行 JavaScript：

```qml
SequentialAnimation {

    NumberAnimation { }

    ScriptAction {
        script: console.log("Done")
    }
}
```

------

# 分类关系

```text
Qt Animation

├── Property Animation
│   ├── NumberAnimation
│   ├── ColorAnimation
│   ├── RotationAnimation
│   └── PropertyAnimation
│
├── Behavior
│
├── Transition
│
├── Animator
│   ├── XAnimator
│   ├── YAnimator
│   ├── OpacityAnimator
│   └── ScaleAnimator
│
└── Animation Group
    ├── SequentialAnimation
    └── ParallelAnimation
```

## 学习建议

建议按以下顺序学习，覆盖绝大多数实际开发场景：

1. **Property Animation**：掌握 `NumberAnimation`、`ColorAnimation`、`PropertyAnimation`。
2. **Behavior**：实现属性变化自动动画，这是 QML 中最常见的动画模式。
3. **Transition + State**：实现界面状态切换动画。
4. **SequentialAnimation / ParallelAnimation**：组合多个动画，构建复杂效果。
5. **Animator**：了解渲染线程动画，在对性能要求较高时使用。

掌握这几部分后，就能够完成绝大多数 Qt Quick 界面动画的开发。