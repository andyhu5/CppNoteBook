windows -> windows 免登

```powershell
# 1. 创建 SSH 系统配置目录（若已存在则跳过）

 New-Item -Path "C:\ProgramData\ssh" -ItemType Directory -Force | Out-Null 
# 2. 将复制的公钥内容写入 administrators_authorized_keys（替换为你的公钥内容）

 "复制的公钥内容" | Out-File -FilePath "C:\ProgramData\ssh\administrators_authorized_keys" -Encoding ASCII 

# 3. 设置文件权限（关键！Windows 需限制权限，否则 SSH 服务器不认） icacls "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r /grant "NT AUTHORITY\SYSTEM:(F)" /grant "BUILTIN\Administrators:(F)"

来自 <https://www.doubao.com/chat/30414265971757570> 
```

