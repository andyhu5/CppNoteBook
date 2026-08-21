## 1. 当前 PowerShell 会话设置代理（临时）

只对当前 PowerShell 窗口有效：

```
$env:HTTP_PROXY="http://127.0.0.1:7890"
$env:HTTPS_PROXY="http://127.0.0.1:7890"
```

检查：

```
echo $env:HTTP_PROXY
echo $env:HTTPS_PROXY
```





## 2. 永久设置 PowerShell 用户环境变量

写入用户环境变量：

```
[Environment]::SetEnvironmentVariable(
    "HTTP_PROXY",
    "http://127.0.0.1:7890",
    "User"
)

[Environment]::SetEnvironmentVariable(
    "HTTPS_PROXY",
    "http://127.0.0.1:7890",
    "User"
)
```

重新打开 PowerShell：

```
echo $env:HTTP_PROXY
```

删除：

```
[Environment]::SetEnvironmentVariable("HTTP_PROXY",$null,"User")
[Environment]::SetEnvironmentVariable("HTTPS_PROXY",$null,"User")
```



如果你想使用 vcpkg 移除所有版本为 **1.90.0#1** 的包，可以先查询，再批量删除。

### 1. 查看已安装的 1.90.0#1 包

```bash
vcpkg list | grep "1.90.0#1"
```

例如输出：

```text
boost-asio:x64-linux        1.90.0#1
boost-system:x64-linux      1.90.0#1
boost-thread:x64-linux      1.90.0#1
```

------

### 2. 批量移除

Linux / WSL：

```bash
vcpkg list | grep "1.90.0#1" | awk '{print $1}' | xargs -r vcpkg remove --recurse
```

说明：

- `awk '{print $1}'` 获取包名：

  ```
  boost-asio:x64-linux
  ```

- `xargs vcpkg remove` 执行删除。

------

### 3. 如果想强制删除，不重新计算依赖

```bash
vcpkg list | grep "1.90.0#1" | awk '{print $1}' | xargs -r vcpkg remove --recurse
```

`--recurse` 会同时删除依赖关系。

------

### 4. Windows PowerShell 写法

如果是在 Windows：

```powershell
vcpkg list | Select-String "1.90.0#1" | ForEach-Object {
    vcpkg remove (($_ -split "\s+")[0]) --recurse
}
```

------

### 5. 删除后确认

```bash
vcpkg list | grep "1.90.0"
```

没有输出表示已经清理。

另外，如果你的目标是解决前面的 `boost_asio 1.90.0 vs 1.91.0` 冲突，更推荐直接：

```bash
vcpkg remove boost-asio:x64-linux --recurse
vcpkg install boost-asio:x64-linux
```

让 vcpkg 重新安装当前 baseline 对应的 Boost 版本。