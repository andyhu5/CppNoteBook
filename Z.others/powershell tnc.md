```powershell
tnc www.baidu.com -p 80



"www.baidu.com","x.com","163.com" | tnc -p 80



$ProgressPreference = 'SilentlyContinue'



"www.baidu.com","x.com","163.com","utools.run" | tnc -p 80 -InformationAction Continue
```







```powershell
"www.baidu.com", "x.com", "163.com", "utools.run" | ForEach-Object -Parallel {
    $result = Test-NetConnection -ComputerName $_ -Port 80
    [PSCustomObject]@{
        Website = $_
        TcpTestSucceeded = $result.TcpTestSucceeded
    }
} | Format-Table -AutoSize
```

