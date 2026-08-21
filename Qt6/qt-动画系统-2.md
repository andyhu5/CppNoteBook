

`State` 和 `Transition` 是 QML 中实现**状态驱动（State-driven UI）**的核心机制。理解它们的关键是：**不要把它们看成动画，而是看成"状态机 + 状态切换动画"。**

可以把它们类比成下面的关系：

```text
State        表示"应该是什么样子"
Transition   表示"如何从一个状态变到另一个状态"
```

例如一个按钮：

```text
Normal
  │
点击
  │
  ▼
Pressed
```

这里：

- `Normal`、`Pressed` 是 **State**
- 从 `Normal` 到 `Pressed` 的变化过程，就是 **Transition**

------

## 一、什么是 State

假设有一个 Rectangle：

```qml
Rectangle {
    width: 100
    height: 100
    color: "red"
}
```

它当前的属性可以理解成：

```text
State ""

width = 100
height = 100
color = red
```

现在希望鼠标点击以后：

```text
width = 300
color = blue
```

State 就可以描述这种目标状态：

```qml
states: [
    State {
        name: "big"

        PropertyChanges {
            target: rect
            width: 300
            color: "blue"
        }
    }
]
```

意思就是：

> 当 state == "big" 时，
> width 应该等于 300，
> color 应该等于 blue。

注意：

**State 描述的是结果，而不是过程。**

------

## 二、什么是 Transition

如果没有 Transition：

```text
width

100 ───────────────► 300
```

立即改变。

加入：

```qml
Transition {
    NumberAnimation {
        properties: "width"
        duration: 500
    }
}
```

以后：

```text
100
110
135
170
220
260
300
```

500ms 内逐渐变化。

所以：

> Transition 描述的是变化过程。

------

## 三、完整流程

例如：

```qml
MouseArea {
    onClicked:
        rect.state = "big"
}
```

整个过程：

```text
初始状态

state=""

width=100

        │
        │ 点击
        ▼

state="big"

PropertyChanges

width=300

        │
        │ Transition
        ▼

100
120
150
180
220
260
300
```

可以看成：

```text
State
        ↓
告诉系统：

"我要变成这样"

Transition
        ↓
告诉系统：

"慢慢变过去"
```

------

# 四、多个 State

例如：

```text
+---------+
| Normal  |
+---------+

      │

+---------+
| Hover   |
+---------+

      │

+---------+
| Pressed |
+---------+

      │

+---------+
| Disabled|
+---------+
```

对应：

```qml
states: [
    State { name: "normal" },

    State {
        name: "hover"

        PropertyChanges {
            target: rect
            scale: 1.05
        }
    },

    State {
        name: "pressed"

        PropertyChanges {
            target: rect
            scale: 0.95
        }
    }
]
```

切换：

```qml
rect.state = "hover"
```

即可。

------

# 五、Transition 可以针对不同状态

例如：

```qml
transitions: [

    Transition {
        from: ""
        to: "big"

        NumberAnimation {
            properties: "width"
            duration: 300
        }
    },

    Transition {
        from: "big"
        to: ""

        NumberAnimation {
            properties: "width"
            duration: 1000
        }
    }
]
```

结果：

```text
Normal ---> Big

300ms

Big ---> Normal

1000ms
```

不同方向可以使用不同动画。

------

# 六、State 不是动画

很多初学者会误以为：

```text
State = Animation
```

其实不是。

应该理解成：

```text
State

描述最终状态

↓

Transition

描述切换动画

↓

Animation

真正执行动画
```

关系图：

```text
            用户点击

                │

                ▼

         rect.state="big"

                │

                ▼

      找到 State "big"

                │

                ▼

      width=300
      color=blue

                │

      有没有 Transition？

           │          │
         没有         有
           │          │
           ▼          ▼

       立即改变     播放动画
```

------

# 七、和 Behavior 的区别（非常重要）

你之前已经学习过 `Behavior`，它和 `Transition` 很容易混淆。

| Behavior             | Transition              |
| -------------------- | ----------------------- |
| 针对单个属性         | 针对状态切换            |
| 任何属性变化都会动画 | 只有切换 `state` 时动画 |
| 不关心状态           | 与 `State` 配合使用     |
| 使用简单             | 适合复杂 UI             |

例如：

```qml
Behavior on width {
    NumberAnimation { duration: 500 }
}
```

以后无论是谁修改：

```qml
width = 100
width = 200
width = 500
```

都会动画。

而：

```qml
Transition {
    NumberAnimation {
        properties: "width"
    }
}
```

只有下面这种情况才会执行：

```qml
rect.state = "big"
```

如果直接写：

```qml
rect.width = 300
```

**Transition 不会生效**。

------

因此，可以用一句话概括两者的职责：

- **State**：定义界面在不同模式下应该具有什么属性值（"目标是什么"）。
- **Transition**：定义界面在不同 State 之间切换时采用什么动画（"如何过去"）。

在实际项目中，像按钮的普通/悬停/按下状态、页面展开与收起、侧边栏显示与隐藏、登录前后界面切换等，都非常适合使用 **State + Transition**；而简单的属性变化（如 `width`、`opacity`、`color`）通常使用 **Behavior** 更直接。