On Windows 11 OpenSSH Server, the default SSH shell is usually **cmd.exe**. You can change it to **PowerShell** by configuring the `DefaultShell` registry key.

### 1. Find PowerShell path

For Windows PowerShell 5.1:

```powershell
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

For PowerShell 7:

```powershell
C:\Program Files\PowerShell\7\pwsh.exe
```

------

### 2. Set SSH default shell

Open **PowerShell as Administrator**:

For PowerShell 7:

```powershell
New-ItemProperty `
    -Path "HKLM:\SOFTWARE\OpenSSH" `
    -Name DefaultShell `
    -Value "C:\Program Files\PowerShell\7\pwsh.exe" `
    -PropertyType String `
    -Force
```

For Windows PowerShell:

```powershell
New-ItemProperty `
    -Path "HKLM:\SOFTWARE\OpenSSH" `
    -Name DefaultShell `
    -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" `
    -PropertyType String `
    -Force
```

------

### 3. Restart SSH service

```powershell
Restart-Service sshd
```

------

### 4. Test

From another machine:

```bash
ssh username@windows11-host
```

You should enter PowerShell directly:

```
Windows PowerShell
Copyright (C) Microsoft Corporation.

PS C:\Users\username>
```

------

### 5. Verify configuration

Check:

```powershell
Get-ItemProperty HKLM:\SOFTWARE\OpenSSH
```

Expected:

```
DefaultShell : C:\Program Files\PowerShell\7\pwsh.exe
```

------

### Optional: Configure only one user

If you do not want to change the global SSH shell, create an SSH user-specific rule in:

```
C:\ProgramData\ssh\sshd_config
```

Example:

```text
Match User andy
    ForceCommand C:\Program Files\PowerShell\7\pwsh.exe
```

Then:

```powershell
Restart-Service sshd
```

However, the registry `DefaultShell` method is the standard approach for Windows 11 OpenSSH.