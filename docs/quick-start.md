# Quick Start Guide

This guide walks you through installing, building, and running Tally for GPU performance isolation.

## Prerequisites

### Hardware Requirements

- Linux x86-64 host
- NVIDIA GPU with Compute Capability 8.0+ (A100, H100, etc.)
- Minimum 32 GB system RAM recommended

### Software Requirements

| Software | Version |
| --- | --- |
| NVIDIA Driver | >= 525 |
| CUDA Toolkit | 12.x |
| CMake | >= 3.24 |
| GCC | >= 10 (C++20 support) |
| Boost | serialization, regex, stacktrace_backtrace |
| Folly | Latest stable |
| Docker | (optional) 20.10+ |

## Installation

### Option 1: Docker (Recommended)

The fastest way to get started:

```bash
# Pull the pre-built image (~130 GB with all dependencies)
docker pull wzhao18/tally:bench

# Launch with GPU access and shared memory
docker run -it \
  --shm-size=64g \
  --gpus all \
  wzhao18/tally:bench /bin/bash
```

The Docker image includes:
- Pre-compiled Tally server and client libraries
- All dependencies (CUDA, Iceoryx, Boost, etc.)
- Benchmark workloads and scripts

### Option 2: Build from Source

```bash
# Clone with submodules
git clone --recursive https://github.com/tally-project/tally.git
cd tally

# Build everything (NCCL + Tally)
make build
```

The build process:
1. Compiles NCCL from `third_party/nccl`
2. Runs CMake configuration
3. Builds all targets: `tally_server`, `tally_client.so`, `tally_client_local.so`, `tally_cutlass.so`

Build output is in the `build/` directory.

#### CMake Options

| Option | Default | Description |
| --- | --- | --- |
| `ENABLE_LOGGING` | OFF | Enable debug logging |
| `ENABLE_PERFORMANCE_LOGGING` | OFF | Enable performance metrics logging |
| `ENABLE_PROFILING` | OFF | Enable profiling instrumentation |
| `VERIFY_CORRECTNESS` | OFF | Verify kernel transformation correctness |
| `MEASURE_PREEMPTION_LATENCY` | OFF | Measure preemption latency |

To enable options:
```bash
cd build
cmake .. -DENABLE_LOGGING=ON -DENABLE_PERFORMANCE_LOGGING=ON
make -j
```

## Running Tally

### Step 1: Start Iceoryx RouDi

The RouDi daemon manages shared memory for IPC:

```bash
./scripts/start_iox.sh
```

This uses the configuration from `config/roudi_config.toml` which defines memory pool sizes.

### Step 2: Start Tally Server

Launch the server with your desired scheduling policy:

```bash
# Basic: no scheduling (pass-through)
./scripts/start_server.sh

# Priority scheduling (recommended for mixed workloads)
SCHEDULER_POLICY=PRIORITY ./scripts/start_server.sh

# Workload-agnostic sharing
SCHEDULER_POLICY=WORKLOAD_AGNOSTIC_SHARING ./scripts/start_server.sh
```

### Step 3: Run Applications with Tally

Use the client script to launch your application with Tally interception:

```bash
# High-priority inference workload (priority=1)
TALLY_CLIENT_PRIORITY=1 ./scripts/start_client.sh python inference_server.py

# Low-priority training workload (priority=0)
TALLY_CLIENT_PRIORITY=0 ./scripts/start_client.sh python train.py
```

### Step 4: Stop Tally

```bash
# Stop the server
./scripts/kill_server.sh

# Stop Iceoryx RouDi
./scripts/kill_iox.sh
```

## Verifying Installation

Run the test suite to verify everything works:

```bash
./scripts/run_test.sh
```

## Common Issues

### "Iceoryx RouDi not running"

Ensure RouDi is started before the server:
```bash
./scripts/start_iox.sh
# Wait a few seconds
./scripts/start_server.sh
```

### "CUDA initialization failed"

Verify GPU access:
```bash
nvidia-smi
# Ensure CUDA_VISIBLE_DEVICES is set correctly
export CUDA_VISIBLE_DEVICES=0
```

### Shared memory errors

If you encounter shared memory pool exhaustion, increase the pool sizes in `config/roudi_config.toml`.

## Next Steps

- [Scheduling Policies](scheduling-policies.md) — Learn about each scheduling strategy
- [Configuration Reference](configuration.md) — Complete environment variable reference
- [Architecture](architecture.md) — Deep dive into system design
