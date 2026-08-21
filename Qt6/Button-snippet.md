

```json
## 水平 垂直
## Button定制
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
    title: qsTr("Hello SQL")

    ColumnLayout {
        anchors.fill: parent
        Rectangle {
            Layout.fillWidth: parent
            Layout.preferredHeight: parent.height - queryBtn.width
            color: "#a3a3a3"
        }
        RowLayout {
            Layout.fillWidth: parent
            Layout.preferredHeight: 100
            Button {
                id:queryBtn
                text:"Query"
                Layout.fillHeight: true

                contentItem: Text {
                    text: queryBtn.text
                    color: "white"          // 设置文字颜色
                    font.family: "Consolas"
                    font.pixelSize: 24
                    horizontalAlignment: Text.AlignHCenter
                    verticalAlignment: Text.AlignVCenter
                    elide: Text.ElideRight
                }

                background: Rectangle {

                    implicitWidth: 120
                    implicitHeight: 50
                    color:"tomato"
                    border.color:"lightyellow"
                    border.width: 5
                    radius: 6
                }
            }

            Button {
                id:insertBtn
                text:"Insert"
                Layout.fillHeight: true

                contentItem: Text {
                    text: insertBtn.text
                    color: "white"          // 设置文字颜色
                    font.family: "Consolas"
                    font.pixelSize: 24
                    horizontalAlignment: Text.AlignHCenter
                    verticalAlignment: Text.AlignVCenter
                    elide: Text.ElideRight
                }

                background: Rectangle {

                    implicitWidth: 120
                    implicitHeight: 50
                    color:"tomato"
                    border.color:"lightyellow"
                    border.width: 5
                    radius: 6
                }
            }
        }
    }

}

```

