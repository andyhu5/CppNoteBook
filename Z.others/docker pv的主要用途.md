pv是什么用途

`pv` 是一个非常实用的命令行工具，全称是 **Pipe Viewer**（管道查看器）。

## 

在这个命令中：

bash

```bash
docker save myimage:tag | pv | ssh user@remote-host 'docker load'
```

`pv` 的作用是**实时显示数据传输的进度**，包括：

### 显示的信息包括：

1. **传输速度**：如 10.5MB/s
2. **已传输数据量**：如 1.2GB
3. **进度条**：可视化显示传输进度
4. **预计剩余时间**：ETA（Estimated Time of Arrival）
5. **已用时间**：传输已经花费的时间

### 实际显示效果示例：

bash

```bash
1.2GiB 0:02:15 [10.5MiB/s] [=========>          ] 45% ETA 0:02:50
```

## 安装 pv

bash

```bash
# Ubuntu/Debian
sudo apt-get install pv

# CentOS/RHEL
sudo yum install pv

# macOS
brew install pv

# Alpine
apk add pv
```