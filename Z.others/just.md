justfile



```
# install just command,for windows 11 :
winget install --id Casey.Just --exact
# for ubuntu
sudo snap install just --classic
```



```makefile
﻿# use PowerShell instead of sh:
set shell := ["powershell.exe","-NoProfile"]

hello:
    Write-Host "Hello, world!"
build:
    go install -ldflags="-s -w" -trimpath
test:
    ping vip.com -n 1
    ping baidu.com -n 1

```

