```json
Column {
    anchors.centerIn: parent
    spacing: 12

    Button {
        text: "Normal"
    }

    Button {
        text: "Flat"
        flat: true
    }

    Button {
        text: "Save"
        highlighted: true
        flat: true
    }

    Button {
        text: "Save"
        font.pixelSize: 20
        padding: 14
        //文字颜色
        palette.buttonText: "#00d0ff"
    }
}
```

```json
//标准的Button
import QtQuick
import QtQuick.Layouts
import QtQuick.Controls.Basic


ApplicationWindow {
    id: window
    width: 640
    height: 480
    minimumWidth: 200
    minimumHeight: 250
    visible: true
    title: qsTr("Hello World")

    Button {
        id:wowButton
        anchors.centerIn: parent
        text: "Launch"
        checkable: false

        background: Rectangle {
            id:bg
            implicitWidth: 180
            implicitHeight: 52
            radius: 12

            color: wowButton.down ?"#0a2a30":"#14343a"
            border.color: "#00e5ff"
            border.width: 2
            scale: wowButton.down ? 0.94:1.0

            Behavior on color {
                ColorAnimation {
                    duration: 150
                }
            }

            Behavior on scale {
                SpringAnimation {
                    spring: 5
                    damping: 0.3
                }
            }
        }

        contentItem: Text {
            text:wowButton.text
            color: "#00e5ff"
            font.pixelSize: 20
            font.bold: true
            horizontalAlignment: Text.AlignHCenter
            verticalAlignment: Text.AlignVCenter
            anchors.fill: parent
        }
    }

}

```

