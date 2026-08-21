import QtQuick
import QtQuick.Layouts
import QtQuick.Controls.Universal

// customCheckBox
// 2026-07-16

ApplicationWindow {
    id: window
    width: 640
    height: 480

    visible: true
    title: qsTr("Hello World")
    
    Flow {
        anchors.fill: parent
        Repeater {
            model: 6
            delegate: CheckBox {
                id: checkbox
                // anchors.centerIn: parent
                text: "One Optional: " + index
    
                indicator: Rectangle {
                    anchors.verticalCenter: parent.verticalCenter
                    width: 24
                    height: 24
                    radius: 6
    
                    border.color: checkbox.checked ? "green" : "gray"
                    color: checkbox.checked ? "lightgreen" : "white"
    
                    Text {
                        anchors.centerIn: parent
                        text: checkbox.checked ? "✅" : ""
                    }
    
                    Behavior on scale {
                        SpringAnimation {
                            spring: 8
                            damping: 0.25
                        }
                    }
    
                    Behavior on color {
    
                        ColorAnimation {
                            from: "tomato"
                            to: "white"
                            duration: 200
                        }
                    }
    
                    Behavior on border.color {
    
                        ColorAnimation {
                            duration: 200
                        }
                    }
    
                    Rectangle {
                        anchors.centerIn: parent
    
                        width: parent.width
                        height: parent.height
    
                        color: "transparent"
                        border.color: "red"
                        border.width: 2
                        opacity: checkbox.checked ? 0.7:0
                        radius: parent.radius
                        scale: checkbox.checked ? 1.45:1.0
                    }
                }
            }
        }
    }
}