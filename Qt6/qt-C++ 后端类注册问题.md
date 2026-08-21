## C++ 后端类注册问题

你的代码有两个问题：

1. `QML_ELEMENT` **不会自动创建一个名为 `TableModel` 的 QML Module**，它只是把类注册到**当前 CMake 定义的 QML Module**。
2. `import TableModel` 要求工程中存在一个 **QML Module**，模块名必须是 `TableModel`。

下面是标准做法（Qt6）。

------

## 方法一：使用 qt_add_qml_module（推荐）

### 1. TableModel.h

你的代码基本不用改。

```cpp
#ifndef TABLEMODEL_H
#define TABLEMODEL_H

#include <QAbstractTableModel>
#include <QQmlEngine>

class TableModel : public QAbstractTableModel
{
    Q_OBJECT
    QML_ELEMENT

public:
    explicit TableModel(QObject *parent = nullptr)
        : QAbstractTableModel(parent)
    {
    }

    int rowCount(const QModelIndex &parent = QModelIndex()) const override
    {
        Q_UNUSED(parent);
        return 200;
    }

    int columnCount(const QModelIndex &parent = QModelIndex()) const override
    {
        Q_UNUSED(parent);
        return 200;
    }

    QVariant data(const QModelIndex &index, int role) const override
    {
        if (!index.isValid())
            return {};

        if (role == Qt::DisplayRole)
            return QString("%1, %2").arg(index.column()).arg(index.row());

        return {};
    }

    QHash<int, QByteArray> roleNames() const override
    {
        return {
            { Qt::DisplayRole, "display" }
        };
    }
};

#endif
```

------

## 2. CMakeLists.txt

最关键的是：

```cmake
qt_add_qml_module(appTableView
    URI TableModel
    VERSION 1.0

    SOURCES
        TableModel.h
)
```

例如：

```cmake
qt_add_executable(appTableView
    main.cpp
)

qt_add_qml_module(appTableView
    URI TableModel
    VERSION 1.0

    QML_FILES
        Main.qml

    SOURCES
        TableModel.h
)
```

这里

```
URI TableModel
```

表示生成一个

```
import TableModel
```

可以导入的模块。

```
QML_ELEMENT
```

表示

```
TableModel
```

这个 C++ 类自动暴露给该模块。

因此 QML 中可以写

```qml
import TableModel

TableModel {
}
```

------

## 3. main.cpp

main.cpp 不需要注册：

```cpp
QQmlApplicationEngine engine;

engine.loadFromModule("TableModel", "Main");
```

如果 Main.qml 不在这个 Module，则保持自己的 URI 即可。

例如：

```cpp
engine.loadFromModule("MyApp", "Main");
```

也没问题。

------

## 方法二：手动注册（不用 QML_ELEMENT）

如果你不想使用 `qt_add_qml_module`，可以在 main.cpp 注册。

删除

```cpp
QML_ELEMENT
```

然后

```cpp
#include <qqml.h>

qmlRegisterType<TableModel>(
    "TableModel",
    1,
    0,
    "TableModel");
```

这样 QML 中同样可以：

```qml
import TableModel 1.0

TableModel {
}
```

这是 Qt5 和 Qt6 都支持的方法。

------

# 为什么你现在会报错？

通常会出现：

```
module "TableModel" is not installed
```

原因是：

你的工程大概率只有

```cmake
qt_add_qml_module(app
    URI MyApp
)
```

而没有

```cmake
URI TableModel
```

因此 Qt 能找到 `MyApp` 模块，却找不到 `TableModel` 模块。

------

# 更推荐的工程组织方式

一般不建议为一个模型单独建立一个 QML Module，而是把所有 C++ 类型放到应用自己的模块中，例如：

```cmake
qt_add_qml_module(app
    URI MyApp
    VERSION 1.0

    SOURCES
        TableModel.h
        Backend.h
        PersonModel.h
)
```

然后：

```qml
import MyApp

ApplicationWindow {

    TableView {
        model: TableModel { }
    }
}
```

这是 Qt 官方示例采用的组织方式，也是大型 Qt 项目最常见的结构。