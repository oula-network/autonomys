# 🤖 Autonomys 挖礦教程 - Linux

## 介紹

Autonomys-farmer 包含以下組件：
- `autonomys-controller` 負責代理 node rpc; 管理集群組件。
- `sharded-cache` piece 分片緩存。
- `full-piece-sharded-cache` piece 分片緩存全量節點。
- `proof-server` GPU 出塊, 計算 proof。
- `plot-server` plotting 服務，encode 數據。
- `plot-client` farming 組件，用於掃盤以及提交 solution。

### 架構

目前所有的集群管理都是基於 nats 來做的, 但是 cache 的具體數據傳輸是通過 TCP 做 p2p 傳輸。

![arch.png](./images/arch.png)

## 軟件和硬件環境建議配置

本軟件僅支持 Linux 操作系統，以及 Nvidia GPU 環境。

### 操作系統及依賴軟件

- Ubuntu 22.04
- GPU 驅動 ≥ 525.60.13 或者直接安裝 cuda 12.4
- 文件系統 Ext4
- Supervisor 4
- Nats-server v2.10.22
- numactl

### 服務器建議配置

| 類別 | CPU | 內存 | GPU | SSD | 網絡 | 運行組件 |
| --- | --- | --- | --- | --- | --- | --- |
| 節點機 | 64核 | 64G/128G | 需要 | 500GiB | 千兆網卡 | `controller` `autonomys-node` `proof-server` `nats-server`  |
| P 盤機 | 每個GPU需要30核 | 每個GPU需要 64 G | 需要 | 20GiB 用於緩存 plot 數據 | 萬兆網卡*2 | `plot-server` `sharded-cache` `full-piece-cache` |
| 存儲機 | 取決於存儲容量 | 取決於存儲容量 | 不需要 | 取決於存儲容量 | 萬兆網卡*2 | `plot-client` |

## 最佳實踐

*注：以下名稱 IP 等都是示例*

### 環境介紹

| 列別 | ip 地址 | 配置 | 部署組件 |
| --- | --- | --- | --- |
| 節點機1 | 192.168.1.1 | GPU*1 | `controller` `autonomys-node` `proof-server` `nats-server`  |
| 節點機2 | 192.168.1.2 | GPU*1 | `controller` `autonomys-node` `proof-server` `nats-server`  |
| 節點機3 | 192.168.1.3 | GPU*1 | `controller` `autonomys-node` `proof-server` `nats-server`  |
| P 盤機1 | 192.168.1.4 | GPU*4 | `autonomys-plot-server-0` `autonomys-plot-server-1` `autonomys-plot-server-2` `autonomys-plot-server-3` `sharded-cache` `full-piece-cache` |
| P 盤機2 | 192.168.1.5 | GPU*4 | `autonomys-plot-server-0` `autonomys-plot-server-1` `autonomys-plot-server-2` `autonomys-plot-server-3` `sharded-cache` `full-piece-cache` |
| 存儲機1 | 192.168.1.6 | 8T NVMe*4 `/mnt/nvme0n1` `/mnt/nvme0n2` `/mnt/nvme1n2` `/mnt/nvme1n1` | `autonomys-plot-client` |
| 存儲機2 | 192.168.1.7 | 8T NVMe*4 `/mnt/nvme0n1` `/mnt/nvme0n2` `/mnt/nvme1n1` `/mnt/nvme1n2` | `autonomys-plot-client` |

### Supervisor 配置

### **節點機配置**

每台節點機需要部署`controller` `autonomys-node` `proof-server` `nats-server` 四個組件。

部署的順序應該是 `nats-server` -> `autonomys-node` -> `controller` -> `proof-server` 。

- **nats-server**

本軟件需要開啟 nats-server jetstream 功能，啟動 nats-server 添加 `--jetstream` falg 即可啟用 jetstream。

nats-server 的配置請參考[nats 官方文檔](https://docs.nats.io/running-a-nats-service/configuration/clustering) 以及 [autonomys nats 配置文檔](https://docs.autonomys.xyz/farming/advanced-cli/cluster#core-messaging-technology-natsio)。

這裡給一個示例 nats-server 配置供參考:
```
server_name=n1-cluster
max_payload = 3MB

jetstream {
   store_dir=/var/nats-data
}


cluster {
  name: c1-cluster
  listen: 0.0.0.0:4248
  routes: [
    nats://192.168.0.1:4248
    nats://192.168.0.2:4248
  ]
}
```

- **autonomys-controller**

```ini
# autonomys-controller 配置
# /etc/supervisor/conf.d/autonomys-controller.conf

[program:autonomys-controller]
command=/root/autonomys/autonomys-farmer cluster --nats-server nats://192.168.1.1:4222 --nats-server nats://192.168.1.2:4222 --nats-server nats://192.168.1.2:4222 controller --tmp --node-rpc-url ws://10.30.1.2:9944
autorestart=true
user=root
redirect_stderr=true
stdout_logfile_maxbytes=100MB
stdout_logfile_backups=2
stdout_logfile=/var/log/autonomys-controller.log
```

- **autonomys-node**
```ini
# autonomys-node 配置
# /etc/supervisor/conf.d/autonomys-node.conf

[program:autonomys-node]
command=/root/autonomys/autonomys-node run --base-path /var/autonomys-node --farmer --rpc-listen-on 0.0.0.0:9944 --chain taurus --sync full --rpc-methods unsafe --rpc-cors all
autorestart=true
user=root
redirect_stderr=true
stdout_logfile_maxbytes=100MB
stdout_logfile_backups=2
stdout_logfile=/var/log/autonomys-node.log
```

- **autonomys-proof-server**
```ini
# autonomys-proof-server 配置
# /etc/supervisor/conf.d/autonomys-proof-server.conf

[program:autonomys-proof-server]
command=/root/autonomys/autonomys-farmer cluster --nats-server nats://192.168.1.1:4222 --nats-server nats://192.168.1.2:4222 --nats-server nats://192.168.1.2:4222 proof-server
autorestart=true
user=root
environment=CUDA_VISIBLE_DEVICES=0
redirect_stderr=true
stdout_logfile_maxbytes=500MB
stdout_logfile_backups=2
stdout_logfile=/var/log/autonomys-proof-server.log
```
啟動命令參數及環境變量解釋解釋:
- --nats-server 參數指定 nats 服務器地址
- `CUDA_VISIBLE_DEVICES` 環境變量指定 GPU, 0  表示 GPU0, 1 表示GPU1, 以此類推。

#### **P 盤機配置 (以 4 GPU為例)**
P 盤機需要部署 `autonomys-plot-server`, `autonomys-sharded-cache`, `autonomys-full-piece-cache` 3 個組件。

`autonomys-plot-server` 組件從 `autonomys-sharded-cache` 和 `autonomys-full-piece-cache` 組件獲取 piece 用於 p 盤。

**autonomys-sharded-cache**
```ini
# sharded-cache 配置
# /etc/supervisor/conf.d/autonomys-sharded-cache.conf

[program:autonomys-sharded-cache]
command=/root/autonomys/autonomys-farmer cluster --nats-server nats://192.168.1.1:4222 --nats-server nats://192.168.1.2:4222 --nats-server nats://192.168.1.2:4222 sharded-cache path=/var/autonomys-sharded-cache
autorestart=true
user=root
redirect_stderr=true
stdout_logfile_maxbytes=100MB
stdout_logfile_backups=2
stdout_logfile=/var/log/autonomys-sharded-cache.log
```
參數解釋:
- --nats-server 參數指定 nats 服務器地址
- path=/path/to/autonomys-sharded-cache 用於指定 piece 緩存存儲路徑


**autonomys-full-piece**
```ini
# autonomys-full-piece 配置
# /etc/supervisor/conf.d/autonomys-full-piece.conf

[program:autonomys-full-piece]
command=/root/autonomys/autonomys-farmer cluster --nats-server nats://192.168.1.1:4222 --nats-server nats://192.168.1.2:4222 --nats-server nats://192.168.1.2:4222 full-piece-sharded-cache --tmp path=/var/autonomys-full-piece
autorestart=true
user=root
redirect_stderr=true
stdout_logfile_maxbytes=100MB
stdout_logfile_backups=2
stdout_logfile=/var/log/autonomys-full-piece.log
```
啟動命令參數解釋:
- --nats-server 參數指定 nats 服務器地址
- path=/path/to/autonomys-full-piece 參數指定 full-piece 存儲路徑


**autonomys-plot-server**
```ini
# autonomys-plot-server 配置文件
# /etc/supervisor/conf.d/autonomys-plot-server.conf

[group:autonomys-plot-server]
programs=autonomys-plot-server-0,autonomys-plot-server-1,autonomys-plot-server-2,autonomys-plot-server-3
[program:autonomys-plot-server-0]
command=numactl -C 0-31 -l /root/autonomys/autonomys-farmer cluster --nats-server nats://192.168.1.1:4222 --nats-server nats://192.168.1.2:4222 --nats-server nats://192.168.1.2:4222 plot-server --priority-cache --listen-port 9966 /var/plot-server/base-path-0
autorestart=true
user=root
environment=CUDA_VISIBLE_DEVICES=0
redirect_stderr=true
stdout_logfile_maxbytes=100MB
stdout_logfile_backups=2
stdout_logfile=/var/log/autonomys-plotter-0.log

[program:autonomys-plot-server-1]
command=numactl -C 96-127 -l /root/autonomys/autonomys-farmer cluster --nats-server nats://192.168.1.1:4222 --nats-server nats://192.168.1.2:4222 --nats-server nats://192.168.1.2:4222 plot-server --priority-cache --listen-port 9967 /var/plot-server/base-path-1
autorestart=true
user=root
environment=CUDA_VISIBLE_DEVICES=1
redirect_stderr=true
stdout_logfile_maxbytes=100MB
stdout_logfile_backups=2
stdout_logfile=/var/log/autonomys-plotter-1.log

[program:autonomys-plot-server-2]
command=numactl -C 96-127 -l /root/autonomys/autonomys-farmer cluster --nats-server nats://192.168.1.1:4222 --nats-server nats://192.168.1.2:4222 --nats-server nats://192.168.1.2:4222 plot-server --priority-cache --listen-port 9968 /var/plot-server/base-path-2
autorestart=true
user=root
environment=CUDA_VISIBLE_DEVICES=2
redirect_stderr=true
stdout_logfile_maxbytes=100MB
stdout_logfile_backups=2
stdout_logfile=/var/log/autonomys-plotter-2.log

[program:autonomys-plot-server-3]
command=numactl -C 144-175 -l /root/autonomys/autonomys-farmer cluster --nats-server nats://192.168.1.1:4222 --nats-server nats://192.168.1.2:4222 --nats-server nats://192.168.1.2:4222 plot-server --priority-cache --listen-port 9969 /var/plot-server/base-path-3
autorestart=true
user=root
environment=CUDA_VISIBLE_DEVICES=3
redirect_stderr=true
stdout_logfile_maxbytes=100MB
stdout_logfile_backups=2
stdout_logfile=/var/log/autonomys-plotter-3.log
```

參數解釋:
- --nats-server 參數指定 nats 服務器地址

環境變量解釋:

- `CUDA_VISIBLE_DEVICES`  指定 GPU, 0  表示 GPU0, 1 表示GPU1, 以此類推。
- `GPU_CONCURRENCY`  增大此值會提高顯存使用量，在使用不同型號的 GPU 時，可以考慮適當調整該變量。

需要注意的是， 使用 numactl 工具綁定 CPU 核心時, 最好也考慮 GPU 的 numa 親和性, 以達到最佳性能。

使用 `nvidia-smi topo -m` 命令查看 GPU numa 親和性。

```bash
# nvidia-smi topo -m
        GPU0    GPU1    NIC0    NIC1    CPU Affinity    NUMA Affinity   GPU NUMA ID
GPU0     X      SYS     NODE    NODE    0-47,96-143     0               N/A
GPU1     X      SYS     NODE    NODE    0-47,96-143     0               N/A
GPU2    SYS      X      SYS     SYS     48-95,144-191   1               N/A
GPU3    SYS      X      SYS     SYS     48-95,144-191   1               N/A
NIC0    NODE    SYS      X      PIX
NIC1    NODE    SYS     PIX      X

Legend:

  X    = Self
  SYS  = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes (e.g., QPI/UPI)
  NODE = Connection traversing PCIe as well as the interconnect between PCIe Host Bridges within a NUMA node
  PHB  = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
  PXB  = Connection traversing multiple PCIe bridges (without traversing the PCIe Host Bridge)
  PIX  = Connection traversing at most a single PCIe bridge
  NV#  = Connection traversing a bonded set of # NVLinks

NIC Legend:

  NIC0: mlx5_0
  NIC1: mlx5_1
```

#### **存儲機配置(以 4 盤為例)**

**autonomys-plot-client**
```ini
# autonomys-plot-client 配置
# /etc/supervisor/conf.d/autonomys-plot-client.conf

[program:autonomys-plot-client]
command=/root/autonomys/autonomys-farmer cluster --nats-server nats://192.168.1.1:4222 --nats-server nats://192.168.1.2:4222 --nats-server nats://192.168.1.2:4222 plot-client --reward-address stBR..S8V  path=/mnt/nvme0n1/,sectors=8000  path=/mnt/nvme0n2/,sectors=8000 path=/mnt/nvme1n0/,sectors=8000 path=/mnt/nvme1n1/,sectors=8000
autorestart=true
user=root
redirect_stderr=true
stdout_logfile_maxbytes=100MB
stdout_logfile_backups=2
stdout_logfile=/var/log/autonomys-plot-client.log
```
啟動命令參數解釋:
- --nats-server 參數指定 nats 服務器地址
- path=/path/to/plot-dir,sectors=8000 指定 plot 文件路徑以及 plot 的扇區數量

## 附錄

### 使用命令

手動初始化集群, 執行後會在n秒後重新初始化整個集群

```shell
autonomys-farmer util \
reinitialization-cache \
    --nats-servers nats://192.168.200.6:4222 \
    --delay 0
```

• --delay 0: 初始化延遲, 單位秒.

模擬 plot 的 download sector 過程, 對 cache cluster 發起請求, 檢查集群狀態

```shell
autonomys-farmer util \
sharded-cache-benchmark \
    --nats-servers nats://192.168.0.2:4222 \
    --sectors 256 \
    --epoch 1 \
    --cache-item-type split-parity-piece
```
