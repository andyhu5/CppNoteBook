List View 和 model的关系

你的代码：

```qml
ListView {
    model: itemDBModel
    delegate: itemListDelegate
}
```

delegate 每创建一次，QML 都会为它提供一组上下文数据。

例如：

```qml
ListModel {
    ListElement {
        _id: 5
        name: "ASM"
        age: 600
    }
}
```

delegate：

```qml
Component {
    id: itemListDelegate

    Rectangle {
        required property int index
        required property int _id
        required property string name
        required property int age
    }
}
```

QML 会自动完成类似下面的绑定（概念上）：

```qml
Rectangle {
    property int index: model.index
    property int _id: model._id
    property string name: model.name
    property int age: model.age
}
```

也就是说：

- `index` —— **不是来自 Model**，而是 `ListView` 自动提供的当前项索引。
- `_id` —— 来自 `itemDBModel` 中每个元素的 `_id`。
- `name` —— 来自 `itemDBModel`。
- `age` —— 来自 `itemDBModel`。

------

### `required property` 的作用

例如：

```qml
required property string name
```

表示：

> delegate 创建时，必须有人给这个属性赋值，否则创建失败。

对于 `ListView` 而言，赋值者就是 `model`。

如果 `itemDBModel` 中没有 `name` 这个角色，就会报：

```text
Required property "name" was not initialized
```

------

### 等价于旧写法

以前很多人会这样写：

```qml
Text {
    text: model.name
}
```

或者：

```qml
Text {
    text: name
}
```

Qt 6 推荐使用 `required property`：

```qml
required property string name

Text {
    text: name
}
```

这样有几个优点：

1. 类型明确（`string`、`int` 等）。
2. 编译器能检查错误。
3. 如果角色不存在，会立即报错，而不是默默得到 `undefined`。

------

### 你的数据是正确的

你之前打印的是：

```text
{"_id":5,"age":600,"name":"ASM"}
{"_id":3,"age":500,"name":"CPP"}
```

因此：

```qml
required property int _id
required property string name
required property int age
```