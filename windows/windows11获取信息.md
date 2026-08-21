| 名称                  | PowerShell位置                                               | 用途                |
| --------------------- | ------------------------------------------------------------ | ------------------- |
| Device ID（设置页面） | `HKLM:\SOFTWARE\Microsoft\SQMClient\MachineId`               | Windows设备标识     |
| MachineGuid           | `HKLM:\SOFTWARE\Microsoft\Cryptography\MachineGuid`          | Windows安装实例标识 |
| Product ID            | `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProductId` | Windows许可证产品ID |
| BIOS UUID             | `Win32_ComputerSystemProduct.UUID`                           | 硬件主板标识        |

例如完整查看：

```powershell
[PSCustomObject]@{
    DeviceID = (Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\SQMClient").MachineId
    MachineGuid = (Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Cryptography").MachineGuid
    ProductID = (Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion").ProductId
    BIOS_UUID = (Get-CimInstance Win32_ComputerSystemProduct).UUID
}
```

输出：

```
DeviceID    : {BA8B90C8-9F2C-4F19-A936-09499E7F6C79}
MachineGuid : 2adaecdc-030c-49a4-83f0-1dba15f84d0d
ProductID   : 00330-80000-00000-AA176
BIOS_UUID   : 03FF0210-04E0-05C2-8006-4A0700080009
```

```powershell
#磁盘信息
Get-CimInstance Win32_DiskDrive | Select-Object Model, SerialNumber
```

```powershell
#网卡Mac地址
Get-NetAdapter |
Where-Object {
     $_.MacAddress -and
     $_.Status -eq "Up" -and
     $_.MacAddress -notlike "00-15*" -and
     $_.MacAddress -notlike "0A-00*"
} |
Select-Object MacAddress
```

