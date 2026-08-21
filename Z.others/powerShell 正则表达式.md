PowerShell 正则表达式

```powershell
ls  |Select-Object -ExpandProperty Name | Select-String "^A.*\.exe$" -CaseSensitive
```



```powershell
PS C:\Users\sheng> ls  |Select-Object -ExpandProperty Name | Select-String "^A.*\.exe$" -CaseSensitive  | gm


   TypeName: Microsoft.PowerShell.Commands.MatchInfo

Name         MemberType Definition
----         ---------- ----------
Equals       Method     bool Equals(System.Object obj)
GetHashCode  Method     int GetHashCode()
GetType      Method     type GetType()
RelativePath Method     string RelativePath(string directory)
ToString     Method     string ToString(), string ToString(string directory)
Context      Property   Microsoft.PowerShell.Commands.MatchInfoContext Context {get;set;}
Filename     Property   string Filename {get;}
IgnoreCase   Property   bool IgnoreCase {get;set;}
Line         Property   string Line {get;set;}
LineNumber   Property   int LineNumber {get;set;}
Matches      Property   System.Text.RegularExpressions.Match[] Matches {get;set;}
Path         Property   string Path {get;set;}
Pattern      Property   string Pattern {get;set;}
```



你的命令主要有两个问题：

1. `Select-String` 返回的是 **MatchInfo 对象**，不是纯字符串，所以后面的 `Replace()` 需要先取 `.Line`。
2. 你替换的是 `"exe"`，会把文件名中任意位置的 `exe` 替换，最好只替换扩展名 `.exe`。

修正：

```powershell
ls | Select-Object -ExpandProperty Name |
    Select-String "^A.*\.exe$" -CaseSensitive |
    ForEach-Object { $_.Line.Replace(".exe", ".com") }
```

输出：

```text
Autoruns.com
Autoruns64.com
Autoruns64a.com
```

------

更 PowerShell 风格的写法：

```powershell
Get-ChildItem |
    Where-Object { $_.Name -cmatch '^A.*\.exe$' } |
    ForEach-Object { $_.Name -replace '\.exe$', '.com' }
```

这里：

```powershell
-replace
```

本身就是 Regex 替换。

解释：

```regex
\.exe$
```

表示：

- `\.`
  - 匹配字面量 `.`
- `exe`
  - 扩展名
- `$`
  - 必须在字符串末尾

所以：

```text
Autoruns.exe
```

变成：

```text
Autoruns.com
```

而不会误伤：

```text
Autoruns.exe.backup
```

------

如果只是文件名处理，第二种更符合 PowerShell 的对象模型：

```
Get-ChildItem
      |
      v
FileInfo对象
      |
      v
Where-Object过滤
      |
      v
-replace修改Name
```

不需要先转换成字符串再用 `Select-String`。