Flickable

```JSON
//一个大尺寸的图片，可以拖动；

Flickable {
    anchors.fill: parent

    // 内容的实际大小
    contentWidth: contentItem.childrenRect.width
    contentHeight: contentItem.childrenRect.height

    Rectangle {
        width: 1024
        height: 768

        gradient: Gradient {
            GradientStop {
                position: 0.0
                color: "yellow"
            }
            GradientStop {
                position: 1.0
                color: "steelblue"
            }
        }

        Text {
            anchors.centerIn: parent
            font.pixelSize: 40
            text: "Drag Me!"
        }
    }
}
```