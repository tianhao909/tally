# 快速开始指南

本指南将引导您完成 Tally 的安装、构建和运行，以实现 GPU 性能隔离。

## 前置条件

### 硬件要求

- Linux x86-64 主机
- NVIDIA GPU，计算能力 8.0+（A100、H100 等）
- 建议最少 32 GB 系统内存

### 软件要求

| 软件 | 版本 |
| --- | --- |
| NVIDIA 驱动 | >= 525 |
| CUDA Toolkit | 12.x |
| CMake | >= 3.24 |
| GCC | >= 10（C++20 支持） |
| Boost | serialization、regex、stacktrace_backtrace |
| Folly | 最新稳定版 |
| Docker | （可选）20.10+ |

## 安装

### 方式一：Docker（推荐）

最快的上手方式：

```bash
# 拉取预构建镜像（约 130 GB，包含所有依赖）
docker pull wzhao18/tally:bench

# 启动容器，挂载 GPU 和共享内存
docker run -it \
  --shm-size=64g \
  --gpus all \
  wzhao18/tally:bench /bin/bash
```

Docker 镜像包含：
- 预编译的 Tally 服务端和客户端库
- 所有依赖项（CUDA、Iceoryx、Boost 等）
- 基准测试工作负载和脚本

### 方式二：从源码编译

```bash
# 克隆仓库（含子模块）
git clone --recursive https://github.com/tally-project/tally.git
cd tally

# 编译所有组件（NCCL + Tally）
make build
```

编译过程：
1. 从 `third_party/nccl` 编译 NCCL
2. 运行 CMake 配置
3. 构建所有目标：`tally_server`、`tally_client.so`、`tally_client_local.so`、`tally_cutlass.so`

编译输出位于 `build/` 目录。

#### CMake 选项

| 选项 | 默认值 | 描述 |
| --- | --- | --- |
| `ENABLE_LOGGING` | OFF | 启用调试日志 |
| `ENABLE_PERFORMANCE_LOGGING` | OFF | 启用性能指标日志 |
| `ENABLE_PROFILING` | OFF | 启用性能分析插桩 |
| `VERIFY_CORRECTNESS` | OFF | 验证内核转换正确性 |
| `MEASURE_PREEMPTION_LATENCY` | OFF | 测量抢占延迟 |

启用选项：
```bash
cd build
cmake .. -DENABLE_LOGGING=ON -DENABLE_PERFORMANCE_LOGGING=ON
make -j
```

## 运行 Tally

### 步骤 1：启动 Iceoryx RouDi

RouDi 守护进程管理 IPC 的共享内存：

```bash
./scripts/start_iox.sh
```

这使用 `config/roudi_config.toml` 中定义的内存池大小配置。

### 步骤 2：启动 Tally 服务端

以所需的调度策略启动服务端：

```bash
# 基本模式：不进行调度（直通）
./scripts/start_server.sh

# 优先级调度（推荐用于混合工作负载）
SCHEDULER_POLICY=PRIORITY ./scripts/start_server.sh

# 工作负载无关共享
SCHEDULER_POLICY=WORKLOAD_AGNOSTIC_SHARING ./scripts/start_server.sh
```

### 步骤 3：使用 Tally 运行应用

使用客户端脚本启动带 Tally 拦截的应用：

```bash
# 高优先级推理工作负载（优先级=1）
TALLY_CLIENT_PRIORITY=1 ./scripts/start_client.sh python inference_server.py

# 低优先级训练工作负载（优先级=0）
TALLY_CLIENT_PRIORITY=0 ./scripts/start_client.sh python train.py
```

### 步骤 4：停止 Tally

```bash
# 停止服务端
./scripts/kill_server.sh

# 停止 Iceoryx RouDi
./scripts/kill_iox.sh
```

## 验证安装

运行测试套件以验证一切正常：

```bash
./scripts/run_test.sh
```

## 常见问题

### "Iceoryx RouDi not running"

确保在启动服务端之前先启动 RouDi：
```bash
./scripts/start_iox.sh
# 等待几秒
./scripts/start_server.sh
```

### "CUDA initialization failed"

验证 GPU 访问：
```bash
nvidia-smi
# 确保正确设置 CUDA_VISIBLE_DEVICES
export CUDA_VISIBLE_DEVICES=0
```

### 共享内存错误

如果遇到共享内存池耗尽，请增大 `config/roudi_config.toml` 中的池大小。

## 下一步

- [调度策略](scheduling-policies_cn.md) — 了解每种调度策略
- [配置参考](configuration_cn.md) — 完整的环境变量参考
- [架构设计](architecture_cn.md) — 深入系统设计
