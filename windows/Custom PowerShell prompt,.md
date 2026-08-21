To include the **hostname** in the PowerShell prompt, customize the `prompt` function in your PowerShell profile.

### 1. Check your PowerShell profile

Run:

```powershell
Test-Path $PROFILE
```

If it returns `False`, create it:

```powershell
New-Item -Path $PROFILE -ItemType File -Force
```

Open the profile:

```powershell
notepad $PROFILE
```

------

### 2. Add a custom prompt

Example:

```powershell
function prompt {
    $hostName = $env:COMPUTERNAME
    $userName = $env:USERNAME
    $path = Get-Location

    "[$userName@$hostName] $path> "
}
```

Restart PowerShell.

The prompt becomes:

```text
[andy@WIN11-PC] C:\Users\andy>
```

------

### 3. For SSH sessions, show hostname automatically

When using SSH:

```bash
ssh andy@win11-pc
```

you will see:

```text
[andy@WIN11-PC] C:\Users\andy>
```

------

### 4. More Linux-style prompt

If you prefer:

```text
andy@WIN11-PC:~>
```

use:

```powershell
function prompt {
    $hostName = $env:COMPUTERNAME
    $userName = $env:USERNAME
    $path = (Get-Location).Path.Replace($HOME,"~")

    "$userName@$hostName`:$path> "
}
```

Example:

```text
andy@DESKTOP-AB123:~/Documents>
```

------

### 5. Include current time (useful for SSH administration)

```powershell
function prompt {
    $time = Get-Date -Format "HH:mm:ss"
    "$time [$env:USERNAME@$env:COMPUTERNAME] $(Get-Location)> "
}
```

Output:

```text
16:32:10 [andy@WIN11-SERVER] C:\Users\andy>
```

The profile location depends on the shell:

- Windows PowerShell 5.1:

  ```
  C:\Users\<user>\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
  ```

- PowerShell 7:

  ```
  C:\Users\<user>\Documents\PowerShell\Microsoft.PowerShell_profile.ps1
  ```

For an SSH server, PowerShell 7 profile is usually the one to modify if your `DefaultShell` is set to `pwsh.exe`.