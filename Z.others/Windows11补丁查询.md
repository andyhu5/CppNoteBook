Windows11补丁查询

```powershell
# 安装 PSWindowsUpdate 模块（需管理员）
Install-Module PSWindowsUpdate -Force -Scope CurrentUser

# 导入模块
Import-Module PSWindowsUpdate

# 然后执行查询
Get-WUHistory | Where-Object { $_.Title -match "Security" }

Get-WUHistory | ? {$_.Title -like "*.NET*"}
```

