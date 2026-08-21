```c
void TableModel::updateValues()
{
    m_count++;


    for(int row = 0; row < m_data.size(); row++)
    {

        // 更新 Age 列
        m_data[row][2] =
            QString::number(25 + row + m_count);


        QModelIndex index =
            createIndex(row, 2);


        emit dataChanged(index,
                         index,
                         {Qt::DisplayRole});

    }

// 1 createIndex是什么意义？
// 2 dataChanged后面为什么有两个index?
```



------

## 1. `createIndex(row, column)` 是什么意义？

`createIndex()` 的作用：

> 根据行号和列号，创建一个 `QModelIndex` 对象，用来唯一定位 Model 中的一个数据单元格。

你的 Model：

```
        column
          0       1        2
        ID      Name     Age
row 0    1       Tom      25
row 1    2       Jack     30
row 2    3       Alice    28
row 3    4       Bob      35
```

例如：

```cpp
QModelIndex index = createIndex(2, 1);
```

表示：

```
row = 2
column = 1
```

也就是：

```
Alice
```

这个 index 内部包含：

```
QModelIndex
 |
 +--- row = 2
 |
 +--- column = 1
 |
 +--- 指向 TableModel
```

之后 Qt 可以通过：

```cpp
data(index, Qt::DisplayRole)
```

知道：

```cpp
return m_data[2][1];
```

得到：

```
Alice
```

------

你的代码：

```cpp
QModelIndex index =
        createIndex(row, 2);
```

因为：

```cpp
column = 2
```

所以定位的是 Age 列。

第一次循环：

```cpp
row = 0
```

创建：

```
(row=0,column=2)
```

对应：

```
25
```

第二次：

```
(row=1,column=2)
```

对应：

```
30
```

------

## 2. 为什么 `dataChanged()` 有两个 index？

函数定义：

```cpp
void dataChanged(
    const QModelIndex &topLeft,
    const QModelIndex &bottomRight,
    const QList<int> &roles = {}
);
```

意思：

> 数据发生变化的区域，从左上角到右下角。

------

例如：

只修改一个单元格：

```
+----+-------+-----+
| ID | Name  | Age |
+----+-------+-----+
| 1  | Tom   | 25  |
+----+-------+-----+
```

修改 Age：

```
(row=0,column=2)
```

那么：

```cpp
emit dataChanged(
    index,
    index,
    {Qt::DisplayRole}
);
```

两个 index 一样：

```
topLeft
   |
   v
(row=0,col=2)


bottomRight
   |
   v
(row=0,col=2)
```

表示：

```
只有一个格子变化
```

------

但是如果一次修改多个区域：

例如：

原来：

```
+----+-------+-----+
| ID | Name  | Age |
+----+-------+-----+
| 1  | Tom   | 25  |
| 2  | Jack  | 30  |
| 3  | Alice | 28  |
+----+-------+-----+
```

Age 三个值全部更新。

可以：

```cpp
QModelIndex topLeft =
    createIndex(0,2);


QModelIndex bottomRight =
    createIndex(2,2);


emit dataChanged(
    topLeft,
    bottomRight,
    {Qt::DisplayRole}
);
```

表示：

```
        Age
        |
        v

25  <-- topLeft
30
28  <-- bottomRight
```

整个区域刷新。

------

## 3. 为什么不用每次一个 index？

你的代码：

```cpp
for(int row = 0; row < m_data.size(); row++)
{
    m_data[row][2] = xxx;


    QModelIndex index =
        createIndex(row,2);


    emit dataChanged(index,index);
}
```

效果：

```
刷新一次
刷新一次
刷新一次
刷新一次
```

4行数据：

```
dataChanged()
dataChanged()
dataChanged()
dataChanged()
```

可以工作。

但是效率一般。

更好的写法：

```cpp
void TableModel::updateValues()
{

    m_count++;


    for(int row=0; row<m_data.size(); row++)
    {
        m_data[row][2] =
            QString::number(25+row+m_count);
    }


    QModelIndex top =
        createIndex(0,2);


    QModelIndex bottom =
        createIndex(
            m_data.size()-1,
            2
        );


    emit dataChanged(
        top,
        bottom,
        {Qt::DisplayRole}
    );
}
```

一次通知：

```
Age列全部刷新
```

------

## 4. 更推荐的理解方式

把 `QModelIndex` 看成：

```
数据库表中的主键定位
```

例如：

```
TableModel

row,column

(3,2)
```

就是：

```
第4行
第3列
```

而：

```cpp
dataChanged(A,B)
```

类似：

SQL：

```sql
UPDATE table
SET age=xxx
WHERE row BETWEEN A AND B
```

------

所以记忆：

| 概念                   | 作用                          |
| ---------------------- | ----------------------------- |
| `createIndex(row,col)` | 创建一个单元格的位置          |
| `QModelIndex`          | Qt内部表示一个数据位置        |
| `data(index,role)`     | 读取这个位置的数据            |
| `dataChanged(a,b)`     | 通知View某个区域的数据变了    |
| 两个index              | 表示刷新范围：左上角 → 右下角 |

Qt Model/View 的设计就是通过这种方式，让大量数据（几万、几十万行）更新时，不需要整个 TableView 重绘。