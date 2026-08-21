`Qt::DisplayRole` 只是 **Qt::ItemDataRole** 枚举中的一个值。

Qt 的 Model/View 架构中，一个单元格（Cell）可以同时拥有多种数据，例如：

- 显示文本
- 图标
- 字体
- 前景色
- 背景色
- 对齐方式
- 提示信息
- 用户自定义数据

这些都通过 **Role** 区分。

## Qt 内置的常用 Role

| Role                            | 值   | 用途                   |
| ------------------------------- | ---- | ---------------------- |
| `Qt::DisplayRole`               | 0    | 显示文本（最常用）     |
| `Qt::DecorationRole`            | 1    | 图标、图片             |
| `Qt::EditRole`                  | 2    | 编辑时的数据           |
| `Qt::ToolTipRole`               | 3    | ToolTip 提示           |
| `Qt::StatusTipRole`             | 4    | 状态栏提示             |
| `Qt::WhatsThisRole`             | 5    | What's This 帮助       |
| `Qt::FontRole`                  | 6    | 字体                   |
| `Qt::TextAlignmentRole`         | 7    | 文本对齐               |
| `Qt::BackgroundRole`            | 8    | 背景颜色               |
| `Qt::ForegroundRole`            | 9    | 前景颜色（字体颜色）   |
| `Qt::CheckStateRole`            | 10   | 复选框状态             |
| `Qt::AccessibleTextRole`        | 11   | 无障碍文本             |
| `Qt::AccessibleDescriptionRole` | 12   | 无障碍描述             |
| `Qt::SizeHintRole`              | 13   | 推荐大小               |
| `Qt::InitialSortOrderRole`      | 14   | 初始排序顺序           |
| `Qt::DisplayPropertyRole`       | 27   | QML 属性               |
| `Qt::DecorationPropertyRole`    | 28   | QML 图标属性           |
| `Qt::ToolTipPropertyRole`       | 29   | QML Tooltip 属性       |
| `Qt::StatusTipPropertyRole`     | 30   | QML StatusTip 属性     |
| `Qt::WhatsThisPropertyRole`     | 31   | QML What's This 属性   |
| `Qt::UserRole`                  | 256  | 用户自定义 Role 起始值 |

------

# 一个例子

假设你的模型：

```cpp
QVariant TableModel::data(const QModelIndex &index, int role) const
{
    switch (role)
    {
    case Qt::DisplayRole:
        return "Andy";

    case Qt::DecorationRole:
        return QIcon(":/images/user.png");

    case Qt::ForegroundRole:
        return QColor(Qt::red);

    case Qt::BackgroundRole:
        return QColor(Qt::yellow);

    case Qt::FontRole:
        return QFont("Microsoft YaHei", 12, QFont::Bold);

    default:
        return {};
    }
}
```

那么同一个单元格：

```
┌────────────┐
│   Andy 👤  │
└────────────┘
```

实际上包含了：

```
DisplayRole      -> "Andy"
DecorationRole   -> 用户头像
ForegroundRole   -> 红色文字
BackgroundRole   -> 黄色背景
FontRole         -> 微软雅黑、粗体
```

------

# UserRole

真正项目中最常用的是 **UserRole**。

例如数据库：

| ID   | Name | Age  |
| ---- | ---- | ---- |
| 100  | Andy | 18   |

界面只显示：

```
Andy
```

但是点击以后，需要知道：

```
id = 100
```

于是：

```cpp
enum Roles
{
    IdRole = Qt::UserRole,
    NameRole,
    AgeRole
};
```

然后：

```cpp
QVariant TableModel::data(const QModelIndex &index, int role) const
{
    const Person &p = persons[index.row()];

    switch(role)
    {
    case IdRole:
        return p.id;

    case NameRole:
        return p.name;

    case AgeRole:
        return p.age;

    default:
        return {};
    }
}
```

对应：

```cpp
QHash<int,QByteArray> roleNames() const
{
    return {
        {IdRole,   "id"},
        {NameRole, "name"},
        {AgeRole,  "age"}
    };
}
```

这样在 QML 中：

```qml
Text {
    text: name
}

Button {
    onClicked: {
        console.log(id)
        console.log(age)
    }
}
```

这里：

```
id
name
age
```

实际上就是：

```
Qt::UserRole     -> id
Qt::UserRole +1  -> name
Qt::UserRole +2  -> age
```

------

## 在 TableView 中

如果你的 `delegate` 是：

```qml
delegate: Rectangle {
    Text {
        text: display
    }
}
```

这里的 `display` 来自：

```cpp
roleNames()
{
    return {
        {Qt::DisplayRole, "display"}
    };
}
```

如果改成：

```cpp
enum Roles
{
    NameRole = Qt::UserRole,
    CostRole
};

QHash<int,QByteArray> roleNames() const
{
    return {
        {NameRole, "name"},
        {CostRole, "cost"}
    };
}
```

那么 QML 中可以直接写：

```qml
Text {
    text: name
}

Text {
    text: cost
}
```

这也是 Qt 官方在 QML 中推荐的方式：**对于自定义模型，优先使用 `Qt::UserRole` 及以上的自定义角色，并通过 `roleNames()` 将它们映射为 QML 可直接访问的属性名。**