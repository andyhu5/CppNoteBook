## 设置QuickStyle

打印环境变量或查看启动参数。例如，如果没有显式指定：

```
QT_QUICK_CONTROLS_STYLE=Material
```

Qt 会根据平台选择默认 Style。

你也可以在程序启动时指定：

```c
#include <QQuickStyle>

int main(int argc, char *argv[])
{
    QQuickStyle::setStyle("Basic");   // 或 Fusion、Material、Universal 等

    QGuiApplication app(argc, argv);
    ...
}
```

不同 Style 的 `Popup` 默认外观不同。