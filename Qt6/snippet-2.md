```c
import QtQuick
import QtQuick.Layouts
import QtQuick.Controls.Material

ApplicationWindow {
    id: window
    width: 640
    height: 480
    minimumWidth: 200
    minimumHeight: 250
    visible: true
    title: qsTr("Hello World")

    Popup {
        id:pupup
        width: 350
        height: 60
        padding: 4 //默认有padding，如要靠左边，需要设置为0；

        background: Rectangle {
            border.width: 1
            border.color: "lightgreen"
            radius: 5
        }

        enter: Transition {
            NumberAnimation {
                property: "opacity"
                from: 0
                to: 1
                duration: 250
            }
        }

        exit: Transition {
            NumberAnimation {
                property: "opacity"
                from: 1
                to: 0
                duration: 250
            }
        }



        RowLayout {
            anchors.fill: parent
            spacing: 0

            Image {
                source: "qrc:/images/WeiyunApp.png"
                Layout.preferredHeight: 50
                Layout.preferredWidth: 50

                fillMode: Image.PreserveAspectFit
            }

            Text {
                //文字靠右边对齐....（#复习#）
                Layout.fillWidth: true

                horizontalAlignment: Text.AlignRight
                verticalAlignment: Text.AlignVCenter


                Layout.rightMargin: 0


                color: "Tomato"
                font.family:"Consolas"
                font.bold: true
                font.pixelSize: 12

                antialiasing: true
                text: "F horizontalAlignment are Text.AlignLeft"
            }
        }
    }


    RowLayout {
        spacing: 0

        anchors.fill: parent

        Rectangle {
            color: "red"
            Layout.preferredWidth: 280
            Layout.preferredHeight: 50

            Layout.leftMargin: 5
            Layout.rightMargin: 5

            radius: 3.5

            RowLayout {
                anchors.fill: parent
                spacing: 15


                Image {
                    anchors.leftMargin: 0
                    source: "qrc:/images/WeiyunApp.png"
                    Layout.preferredHeight: 50
                    Layout.preferredWidth: 50
                    fillMode: Image.PreserveAspectFit
                }

                Text {
                    // !!! 不能出现RowLayou和anchors混用的情况...
                    // anchors.rightMargin: 0
                    // anchors.verticalCenter: parent.verticalCenter
                    // anchors.left: parent.left
                    // anchors.leftMargin: 55

                    Layout.alignment: Qt.AlignVCenter

                    color: "white"
                    font:"Consolas"

                    antialiasing: true
                    text: "Hello world"
                }

                Timer {
                    id:popupTimer
                    interval: 2000
                    repeat: false
                    onTriggered: pupup.close()

                }

                Button {
                    id:btn
                    // anchors.fill: parent
                    // width: 50
                    contentItem: Label {
                        color: "red"
                        text: "Click Me"
                        font.family: "Consolas"
                        horizontalAlignment: Text.AlignHCenter
                        verticalAlignment: Text.AlignVCenter
                    }
                    onClicked: {
                        popupTimer.start()
                        pupup.open()
                    }
                }
            }
        }

        Rectangle {
            color: "blue"
            Layout.fillWidth: true
            Layout.preferredHeight: 50
            Layout.rightMargin: 5
        }
    }


}

```

