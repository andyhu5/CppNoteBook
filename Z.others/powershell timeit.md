```powershell
function time($block) {
    $sw = [Diagnostics.Stopwatch]::StartNew()
    &$block
    $sw.Stop()
    $sw.Elapsed
}

.\do_something.ps1  
$command = Get-History -Count 1  
$command.EndExecutionTime - $command.StartExecutionTime
    
Measure-Command { .\do_something.ps1 }
    
```

