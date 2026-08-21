# `activeFocusControl` 属性详解（ApplicationWindow）

## 基本信息

```qml
readonly property Control activeFocusControl
```

`activeFocusControl` 是 **`ApplicationWindow`**（以及 `Window`）提供的一个**只读（read-only）属性**，类型为 **`Control`**。

## 作用是什么？

它表示**当前窗口中拥有键盘焦点（active focus）的那个 `Control` 控件**。

- 当用户通过 Tab 键切换焦点、鼠标点击某个控件、或代码中调用 `forceActiveFocus()` 时，这个属性会自动更新，指向当前获得焦点的控件。
- 如果当前获得焦点的**不是**一个 `Control`（比如是普通的 `Item` 或 `MouseArea`），则 `activeFocusControl` 为 `null`。

## 为什么类型是 `Control` 而不是 `Item` 或 `var`？

这正好可以和前面讨论的 `Item`（footer）与 `var`（gradient）做对比：

| 属性                 | 类型      | 说明                                                         |
| -------------------- | --------- | ------------------------------------------------------------ |
| `footer`             | `Item`    | 任何可视化项都可以                                           |
| `gradient`           | `var`     | 完全灵活，任意值                                             |
| `activeFocusControl` | `Control` | **更严格**——必须是 `Control` 或其子类（Button、TextField、Slider…） |

选择 `Control` 类型是因为：

1. **语义精确**：只有 `Control`（来自 `QtQuick.Controls`）才有"控件"的语义——比如按钮、输入框、下拉框等可交互 UI 组件。普通 `Rectangle` 或 `Item` 即便能获得焦点，也不算"控件"。
2. **暴露特定 API**：`Control` 类型有一些 `Item` 没有的属性/方法，比如 `enabled`、`hovered`、`padding`、`background`、`contentItem` 等，这样使用者可以直接访问这些控件特有的属性。
3. **只读（readonly）**：因为这个值应由 Qt 焦点系统自动管理，不允许开发者手动赋值，只能通过调用控件的 `forceActiveFocus()` 间接改变它。

## 使用示例

```qml
ApplicationWindow {
    id: window
    visible: true

    ColumnLayout {
        TextField { id: field1; placeholderText: "姓名" }
        TextField { id: field2; placeholderText: "邮箱" }
        Button { text: "提交" }
    }

    // 监听焦点变化
    onActiveFocusControlChanged: {
        if (activeFocusControl)
            console.log("当前焦点控件：" + activeFocusControl)
        else
            console.log("当前没有 Control 获得焦点")
    }
}
```

## 与 `activeFocusItem` 的区别

Qt 中还有一个类似但更底层的属性：

| 属性                                                      | 类型      | 范围                                                 |
| --------------------------------------------------------- | --------- | ---------------------------------------------------- |
| `activeFocusItem`（Window 提供）                          | `Item`    | 任何获得焦点的 **Item**（更宽泛）                    |
| `activeFocusControl`（ApplicationWindow / Controls 提供） | `Control` | 仅当焦点项是 **Control** 子类时才有值，否则为 `null` |

也就是说：

- `activeFocusItem` 几乎总是有值（只要有元素获得焦点）；
- `activeFocusControl` **可能是 `null`**——比如焦点落在一个自定义的、非 `Control` 派生的 `Item` 上时。

## 小结

- **类型定为 `Control`** 是为了保证只有真正的"控件"才能赋值给这个属性，同时暴露 Control 特有的 API（如 `enabled`、`background` 等）。
- **只读属性**，由 Qt Quick Controls 的焦点系统自动维护，开发者只能"读取"或"监听变化"，不能直接赋值。
- 这体现了 QML 类型系统的设计哲学：**类型越具体，语义越明确，API 越丰富，但灵活性也越受限**——与 `var`（完全灵活）和 `Item`（中等灵活）形成鲜明对比。