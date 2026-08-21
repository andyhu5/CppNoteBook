



首先，这一句：

```
Image {
    anchors.leftMargin: 1
    anchors.verticalCenter: parent.verticalCenter

    Layout.preferredHeight: 50
    Layout.preferredWidth: 50
}
```

实际上：

- `anchors.leftMargin` **不会生效**
- `anchors.verticalCenter` **也不会生效**

因为 **`Image` 已经由 `RowLayout` 管理**。

Qt 官方的原则是：

> **一个 Item 要么由 Layout 管理，要么由 anchors 管理，不应同时使用两者。**

