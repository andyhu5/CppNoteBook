在 Visual Studio 2022 中，可以通过 **Post-Build Event（生成后事件）** 实现：编译完成后自动把 `.exe` 复制到 D 盘。

你截图里配置的是 **Custom Build Step**，它不是最佳选择。建议使用 **Build Events → Post-Build Event**。

操作步骤：

### 1. 打开项目属性

右键项目：

```
Properties
```

选择：

```
Configuration Properties
    └── Build Events
          └── Post-Build Event
```

------

### 2. 在 Command Line 输入：

例如复制到 `D:\Test`：

```bat
if not exist D:\Test mkdir D:\Test
copy "$(TargetPath)" "D:\Test\"
```

说明：

- `$(TargetPath)`
  Visual Studio 内置宏，表示生成后的完整 exe 路径。

例如：

```
C:\project\x64\Release\MyApp.exe
```

最终执行：

```bat
copy "C:\project\x64\Release\MyApp.exe" "D:\Test\"
```

------

### 3. 设置执行条件

下面：

```
Use In Build
```

保持：

```
Yes
```

即可。

------

### 如果想复制整个 Release 目录

例如：

```
x64\Release\
```

复制到：

```
D:\MyApp\
```

可以：

```bat
xcopy "$(TargetDir)" "D:\MyApp\" /E /Y /I
```

参数：

| 参数 | 含义             |
| ---- | ---------------- |
| `/E` | 包含子目录       |
| `/Y` | 覆盖不询问       |
| `/I` | 目标作为目录处理 |

------

### 如果只复制 exe + dll

常见 C++ 项目：

```bat
copy "$(TargetPath)" "D:\MyApp\"
copy "$(TargetDir)*.dll" "D:\MyApp\"
```

------

### Debug / Release 分开

例如：

Release：

```
D:\Release\
```

Debug：

```
D:\Debug\
```

可以使用：

```bat
if "$(Configuration)"=="Release" (
    copy "$(TargetPath)" "D:\Release\"
)

if "$(Configuration)"=="Debug" (
    copy "$(TargetPath)" "D:\Debug\"
)
```

------

### 你的截图里的 Custom Build Step 不推荐

你现在：

```
Custom Build Step
    Command Line:
    copy %(FullPath) d:\
```

问题：

1. `%(FullPath)` 是 MSBuild Item 元数据，不一定指向最终 exe。
2. Custom Build Step 默认执行阶段不明确。
3. 容易导致每次编译都触发无意义操作。

Visual Studio 推荐流程：

```
Compile
   |
   v
Link
   |
   v
Post-Build Event
   |
   v
Copy exe
```

所以使用：

```
Configuration Properties
    → Build Events
        → Post-Build Event
```

更符合 VS 工程管理方式。