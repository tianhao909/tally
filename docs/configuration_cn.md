# 配置参考

Tally 所有环境变量和配置文件的完整参考。

## 环境变量

### 服务端核心

| 变量 | 默认值 | 类型 | 描述 |
| --- | --- | --- | --- |
| `SCHEDULER_POLICY` | `NAIVE` | String | 调度策略：`NAIVE`、`PROFILE`、`PRIORITY`、`WORKLOAD_AGNOSTIC_SHARING`、`WORKLOAD_AWARE_SHARING` |
| `KERNEL_PROFILE_ITERATIONS` | `5` | uint32 | 内核性能分析的迭代次数 |

### 优先级调度器

| 变量 | 默认值 | 类型 | 描述 |
| --- | --- | --- | --- |
| `PRIORITY_MAX_ALLOWED_PREEMPTION_LATENCY_MS` | `0.1` | float | 最大可容忍的抢占延迟（毫秒） |
| `PRIORITY_MIN_WAIT_TIME_MS` | `0.1` | float | 让步 GPU 给低优先级前的最小等待时间 |
| `PRIORITY_PTB_MAX_NUM_THREADS_PER_SM` | `1024` | uint32 | PTB 内核每个 SM 分配的最大线程数 |
| `PRIORITY_MIN_WORKER_BLOCKS` | `24` | uint32 | 工作内核的最小线程块数 |
| `PRIORITY_FALL_BACK_TO_ORIGINAL_THRESHOLD` | `0.1` | float | 回退到原始配置的性能下降阈值 |
| `PRIORITY_FALL_BACK_TO_ORIGINAL_THRESHOLD_FOR_SHORT_KERNEL` | `0.5` | float | 短运行内核的回退阈值 |
| `PRIORITY_PTB_PREEMPTION_LATENCY_CALCULATION_FACTOR` | `2.0` | float | 计算预期抢占延迟的系数 |
| `PRIORITY_USE_ORIGINAL_CONFIGS` | `FALSE` | bool | 始终使用原始启动配置（禁用转换） |
| `PRIORITY_USE_SPACE_SHARE` | `FALSE` | bool | 启用空间共享（不同优先级在不同 SM 上） |
| `PRIORITY_DISABLE_TRANSFORMATION` | `FALSE` | bool | 完全禁用内核转换 |
| `PRIORITY_SPACE_SHARE_MAX_SM_PERCENTAGE` | `0.0` | float | 空间共享中分配给低优先级的最大 SM 比例 |
| `PRIORITY_WAIT_TIME_MS_TO_USE_ORIGINAL_CONFIGS` | `100.0` | float | 切换到原始配置的等待时间阈值 |

### 共享调度器

| 变量 | 默认值 | 类型 | 描述 |
| --- | --- | --- | --- |
| `SHARING_PTB_MAX_NUM_THREADS_PER_SM` | `1024` | uint32 | 共享模式下每个 SM 的最大线程数 |
| `SHARING_USE_PTB_THRESHOLD` | `0.9` | float | 触发 PTB 转换的占用率阈值 |

### 客户端配置

| 变量 | 默认值 | 类型 | 描述 |
| --- | --- | --- | --- |
| `TALLY_CLIENT_PRIORITY` | `0` | int32 | 客户端优先级（越高越重要） |

## 配置文件

### Iceoryx RouDi 配置

文件：`config/roudi_config.toml`

定义 IPC 通信的共享内存池大小：

```toml
[general]
version = 1

[[segment]]

[[segment.mempool]]
size = 128         # 小消息（少参数的 API 调用）
count = 10000

[[segment.mempool]]
size = 1024        # 中等消息
count = 5000

[[segment.mempool]]
size = 16384       # 内核参数
count = 1000

[[segment.mempool]]
size = 131072      # 大型内核数据
count = 200

[[segment.mempool]]
size = 524288      # 超大数据
count = 50

[[segment.mempool]]
size = 1048576     # 1 MB 块
count = 30

[[segment.mempool]]
size = 4194304     # 4 MB 块
count = 10

[[segment.mempool]]
size = 1073741824  # 1 GB 块（用于大型内存传输）
count = 2

[[segment.mempool]]
size = 3221225472  # 3 GB 块
count = 2
```

**调优指南：**
- 如果看到"pool exhausted"错误，增加小/中型池的 `count`
- 如果通过 Tally 传输非常大的张量，增加大型池的大小
- Docker 的 `--shm-size` 标志必须能容纳总池内存

### Cubin 缓存

位置：`~/.cache/tally/transform/`

缓存存储：
- `uid_count.txt` — 缓存 cubin 的全局唯一 ID 计数器
- `<size>.cache` — 按 cubin 二进制大小索引的序列化 cubin 数据

缓存自动管理：
- 服务端关闭时缓存新的转换结果
- 服务端启动时加载缓存
- 删除该目录可强制重新转换所有内核

## 配置示例

### 最小优先级配置

```bash
SCHEDULER_POLICY=PRIORITY \
./scripts/start_server.sh
```

### 激进抢占（低延迟）

```bash
SCHEDULER_POLICY=PRIORITY \
PRIORITY_MAX_ALLOWED_PREEMPTION_LATENCY_MS=0.05 \
PRIORITY_MIN_WAIT_TIME_MS=0.05 \
./scripts/start_server.sh
```

### 带优先级的空间共享

```bash
SCHEDULER_POLICY=PRIORITY \
PRIORITY_USE_SPACE_SHARE=TRUE \
PRIORITY_SPACE_SHARE_MAX_SM_PERCENTAGE=0.25 \
./scripts/start_server.sh
```

### 简单多租户共享

```bash
SCHEDULER_POLICY=WORKLOAD_AGNOSTIC_SHARING \
SHARING_PTB_MAX_NUM_THREADS_PER_SM=768 \
./scripts/start_server.sh
```
