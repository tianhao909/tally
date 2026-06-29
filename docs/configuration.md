# Configuration Reference

Complete reference for all Tally environment variables and configuration files.

## Environment Variables

### Server Core

| Variable | Default | Type | Description |
| --- | --- | --- | --- |
| `SCHEDULER_POLICY` | `NAIVE` | String | Scheduling policy: `NAIVE`, `PROFILE`, `PRIORITY`, `WORKLOAD_AGNOSTIC_SHARING`, `WORKLOAD_AWARE_SHARING` |
| `KERNEL_PROFILE_ITERATIONS` | `5` | uint32 | Number of iterations for kernel profiling |

### Priority Scheduler

| Variable | Default | Type | Description |
| --- | --- | --- | --- |
| `PRIORITY_MAX_ALLOWED_PREEMPTION_LATENCY_MS` | `0.1` | float | Maximum tolerable preemption latency in milliseconds |
| `PRIORITY_MIN_WAIT_TIME_MS` | `0.1` | float | Minimum time to wait before yielding GPU to lower priority |
| `PRIORITY_PTB_MAX_NUM_THREADS_PER_SM` | `1024` | uint32 | Max threads per SM allocated to PTB kernels |
| `PRIORITY_MIN_WORKER_BLOCKS` | `24` | uint32 | Minimum thread blocks for worker kernels |
| `PRIORITY_FALL_BACK_TO_ORIGINAL_THRESHOLD` | `0.1` | float | Performance degradation threshold to fall back to original config |
| `PRIORITY_FALL_BACK_TO_ORIGINAL_THRESHOLD_FOR_SHORT_KERNEL` | `0.5` | float | Fall-back threshold for short-running kernels |
| `PRIORITY_PTB_PREEMPTION_LATENCY_CALCULATION_FACTOR` | `2.0` | float | Factor for calculating expected preemption latency |
| `PRIORITY_USE_ORIGINAL_CONFIGS` | `FALSE` | bool | Always use original launch configs (disable transformation) |
| `PRIORITY_USE_SPACE_SHARE` | `FALSE` | bool | Enable spatial sharing (different priorities on different SMs) |
| `PRIORITY_DISABLE_TRANSFORMATION` | `FALSE` | bool | Completely disable kernel transformation |
| `PRIORITY_SPACE_SHARE_MAX_SM_PERCENTAGE` | `0.0` | float | Max fraction of SMs allocated to low-priority in spatial sharing |
| `PRIORITY_WAIT_TIME_MS_TO_USE_ORIGINAL_CONFIGS` | `100.0` | float | Wait time threshold to switch to original configs |

### Sharing Scheduler

| Variable | Default | Type | Description |
| --- | --- | --- | --- |
| `SHARING_PTB_MAX_NUM_THREADS_PER_SM` | `1024` | uint32 | Max threads per SM for sharing mode |
| `SHARING_USE_PTB_THRESHOLD` | `0.9` | float | Occupancy threshold to trigger PTB transformation |

### Client Configuration

| Variable | Default | Type | Description |
| --- | --- | --- | --- |
| `TALLY_CLIENT_PRIORITY` | `0` | int32 | Client priority (higher = more important) |

## Configuration Files

### Iceoryx RouDi Configuration

File: `config/roudi_config.toml`

Defines shared memory pool sizes for IPC communication:

```toml
[general]
version = 1

[[segment]]

[[segment.mempool]]
size = 128         # Small messages (API calls with few params)
count = 10000

[[segment.mempool]]
size = 1024        # Medium messages
count = 5000

[[segment.mempool]]
size = 16384       # Kernel parameters
count = 1000

[[segment.mempool]]
size = 131072      # Large kernel data
count = 200

[[segment.mempool]]
size = 524288      # Very large data
count = 50

[[segment.mempool]]
size = 1048576     # 1 MB blocks
count = 30

[[segment.mempool]]
size = 4194304     # 4 MB blocks
count = 10

[[segment.mempool]]
size = 1073741824  # 1 GB blocks (for large memory transfers)
count = 2

[[segment.mempool]]
size = 3221225472  # 3 GB blocks
count = 2
```

**Tuning guidance:**
- Increase `count` for small/medium pools if you see "pool exhausted" errors
- Increase large pool sizes if transferring very large tensors through Tally
- The `--shm-size` Docker flag must accommodate total pool memory

### Cubin Cache

Location: `~/.cache/tally/transform/`

The cache stores:
- `uid_count.txt` — Global unique ID counter for cached cubins
- `<size>.cache` — Serialized cubin data indexed by cubin binary size

The cache is automatically managed:
- New transformations are cached on server shutdown
- Cache is loaded on server startup
- Delete the directory to force re-transformation of all kernels

## Example Configurations

### Minimal Priority Setup

```bash
SCHEDULER_POLICY=PRIORITY \
./scripts/start_server.sh
```

### Aggressive Preemption (Low Latency)

```bash
SCHEDULER_POLICY=PRIORITY \
PRIORITY_MAX_ALLOWED_PREEMPTION_LATENCY_MS=0.05 \
PRIORITY_MIN_WAIT_TIME_MS=0.05 \
./scripts/start_server.sh
```

### Spatial Sharing with Priority

```bash
SCHEDULER_POLICY=PRIORITY \
PRIORITY_USE_SPACE_SHARE=TRUE \
PRIORITY_SPACE_SHARE_MAX_SM_PERCENTAGE=0.25 \
./scripts/start_server.sh
```

### Simple Multi-tenant Sharing

```bash
SCHEDULER_POLICY=WORKLOAD_AGNOSTIC_SHARING \
SHARING_PTB_MAX_NUM_THREADS_PER_SM=768 \
./scripts/start_server.sh
```
