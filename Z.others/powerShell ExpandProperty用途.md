`Select-Object -ExpandProperty` 是 PowerShell 中用于**展开对象的某一个属性，只输出属性值本身**的功能。

PowerShell 和传统 Shell（bash、cmd）最大的区别是：

> PowerShell 管道传递的是 .NET 对象，不是文本。

------

例如：

```powershell
ls
```

实际上执行的是：

```powershell
Get-ChildItem
```

返回的是一组 `FileInfo` 对象。

一个文件对象大概包含：

```text
Name
Length
CreationTime
LastWriteTime
FullName
Extension
Directory
...
```

例如：

```powershell
ls | Select-Object Name
```

输出：

```
Name
----
tcpvcon.exe
tcpview.exe
```

这里返回的仍然是一个对象，只不过对象只有一个属性：

```text
Name = tcpvcon.exe
```

------

而：

```powershell
ls | Select-Object -ExpandProperty Name
```

表示：

> 取出 Name 属性的值，并把它展开成字符串输出。

结果：

```
tcpvcon.exe
tcpview.exe
```

实际管道中传递的是：

```text
"tcpvcon.exe"
"tcpview.exe"
```

而不是：

```text
FileInfo{Name="tcpvcon.exe"}
```

------

举个例子：

假设：

```powershell
$files = ls
```

第一个元素：

```powershell
$files[0]
```

得到：

```
Mode                 LastWriteTime      Length Name
----                 -------------      ------ ----
-a----               4/11/2023          202632 tcpvcon.exe
```

它是：

```text
System.IO.FileInfo
```

对象。

查看类型：

```powershell
($files[0]).GetType()
```

输出：

```
IsPublic IsSerial Name
-------- -------- ----
True     True     FileInfo
```

------

如果：

```powershell
ls | Select-Object Name
```

类型变成：

```
Selected.System.IO.FileInfo
```

还是对象。

如果：

```powershell
ls | Select-Object -ExpandProperty Name
```

类型：

```powershell
(ls | Select-Object -ExpandProperty Name)[0].GetType()
```

结果：

```
System.String
```

------

所以你前面的命令：

```powershell
ls | Select-Object -ExpandProperty Name | grep -E "^t.*\.exe$"
```

流程：

```
Get-ChildItem
        |
        v
FileInfo对象
        |
        v
Select-Object -ExpandProperty Name
        |
        v
字符串:
tcpvcon.exe
tcpview.exe
        |
        v
grep 正则过滤
```

如果不加 `-ExpandProperty`：

```powershell
ls | Select-Object Name | grep -E "^t.*\.exe$"
```

`grep` 接收到的是 PowerShell 格式化后的对象文本，可能包含：

```
Name
----
tcpvcon.exe
tcpview.exe
```

容易受到格式化影响。

------

类似功能还有：

```powershell
Select-Object -Property Name
```

和：

```powershell
Select-Object -ExpandProperty Name
```

区别：

| 命令                                 | 返回                   |
| ------------------------------------ | ---------------------- |
| `Select-Object Name`                 | 对象（只有 Name 属性） |
| `Select-Object -ExpandProperty Name` | Name 属性的值          |

这也是 PowerShell 里经常和 Linux 管道思维不同的地方。Linux：

```
文本 -> 文本 -> 文本
```

PowerShell：

```
对象 -> 对象 -> 对象 -> 最后才格式化成文本
```

所以在 PowerShell 里调用 `grep`（实际上通常是 `Select-String`）前，经常需要先 `-ExpandProperty` 把对象转换成字符串。