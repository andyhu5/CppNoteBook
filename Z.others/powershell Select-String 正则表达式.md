是的，`Select-String` 原生支持正则表达式（regex）。

实际上，`Select-String` 就是 PowerShell 版本的 `grep`，默认使用 **.NET 正则表达式引擎（.NET Regex）**。

------

## 基本用法

例如查找以 `t` 开头、`.exe` 结尾：

```powershell
ls -Name | Select-String "^t.*\.exe$"
```

结果：

```
tcpvcon.exe
tcpvcon64.exe
tcpvcon64a.exe
tcpview.exe
tcpview64.exe
tcpview64a.exe
```

这里：

```regex
^t.*\.exe$
```

就是标准 .NET Regex。

------

## Select-String 默认就是正则模式

例如：

```powershell
"hello123" | Select-String "\d+"
```

输出：

```
hello123
```

因为：

```regex
\d+
```

匹配数字。

------

## 使用 -Pattern

完整写法：

```powershell
Select-String -Pattern "^\d+$"
```

等价于：

```powershell
Select-String "^\d+$"
```

------

## 如果不想使用正则

使用：

```powershell
-SimpleMatch
```

例如：

```powershell
"hello.*" | Select-String -SimpleMatch ".*"
```

匹配字符串：

```
hello.*
```

而不是：

```
hello + 任意字符
```

区别：

| 参数           | 匹配方式   |
| -------------- | ---------- |
| 默认           | Regex      |
| `-SimpleMatch` | 普通字符串 |

------

## 大小写

默认：

```powershell
"ABC" | Select-String "abc"
```

可以匹配。

因为 .NET Regex 默认：

```text
IgnoreCase
```

如果区分大小写：

```powershell
"ABC" | Select-String "abc" -CaseSensitive
```

不会匹配。

------

## 多个模式

例如：

```powershell
ls -Name | Select-String "tcp|udp"
```

匹配：

```
tcpvcon.exe
udpclient.exe
```

------

## 和 Linux grep 对比

| 功能       | grep      | PowerShell |
| ---------- | --------- | ---------- |
| 正则       | 支持      | 支持       |
| 默认大小写 | 敏感      | 不敏感     |
| 数字       | `[0-9]`   | `\d`       |
| 单词       | 无标准    | `\w`       |
| 空白       | 无标准    | `\s`       |
| PCRE       | `grep -P` | .NET Regex |

------

## 一个 PowerShell 常用写法

你的例子：

```powershell
Get-ChildItem -Name | Select-String "^t.*\.exe$"
```

或者更 PowerShell 风格：

```powershell
Get-ChildItem -Filter "t*.exe" -Name
```

如果需要复杂条件：

```powershell
Get-ChildItem |
    Where-Object { $_.Name -match '^t.*\.exe$' }
```

其中：

```powershell
-match
```

也是调用 .NET Regex。

例如：

```powershell
"tcpview64.exe" -match '^t.*\.exe$'
```

返回：

```
True
```

所以 PowerShell 中处理正则通常有三个选择：

1. `Select-String` —— 类似 grep，处理文本流/文件内容
2. `-match` —— 对象属性过滤
3. `Where-Object` + regex —— 管道对象过滤

对于 `Get-ChildItem` 这种对象场景，通常推荐 `-match` 或 `Where-Object`，而不是模拟 grep。