# Kernel Transformation Design

This document describes the technical design of Tally's kernel transformation engine.

## Overview

Tally transforms CUDA kernels at the PTX (Parallel Thread Execution) intermediate representation level to enable fine-grained scheduling control without modifying application source code.

## Persistent Thread Block (PTB) Transformation

### Concept

Standard CUDA kernels launch a fixed number of thread blocks that execute to completion. PTB transformation rewrites kernels to use a persistent set of thread blocks that fetch work from a shared counter, enabling:

- **Controlled occupancy**: Launch fewer blocks than the original, leaving SM resources for other workloads
- **Dynamic work distribution**: Blocks self-schedule by atomically incrementing a global index
- **Preemption support**: Blocks can check a retreat flag and exit early

### PTB Variants

| Variant | Description |
| --- | --- |
| Static PTB | Fixed blocks_per_sm, blocks run to completion |
| Dynamic PTB | Blocks fetch work items from global counter |
| Preemptive PTB | Blocks periodically check a retreat flag |

### Implementation

The transformation modifies kernel grid dimensions and adds control flow:

```
Original: kernel<<<N_blocks, block_size>>>()
PTB:      kernel_ptb<<<blocks_per_sm * num_SMs, block_size>>>(global_idx, retreat_flag)
```

Inside the transformed kernel:
```
while (true) {
    idx = atomicAdd(global_idx, 1)
    if (idx >= N_blocks) break
    if (retreat_flag) break  // preemptive variant only
    // Execute original kernel body with remapped block index
}
```

## Kernel Slicing

### Concept

For kernels where PTB preemption latency is unacceptable, kernel slicing divides the grid into independent sub-grids:

```
Original: kernel<<<(Gx, Gy, Gz), block_size>>>()
Sliced:   for i in 0..N_slices:
              kernel<<<slice_grid, block_size>>>(block_offset[i])
```

### Benefits

- Preemption at slice boundaries (no in-kernel modification needed)
- Works for kernels that cannot be PTB-transformed
- Configurable slice granularity

## Sync-Aware Transformation

### Problem

When launching fewer thread blocks than the original kernel (via PTB), `__syncthreads()` barriers can deadlock because not all expected blocks are present.

### Solution

The sync-aware transformation:
1. Identifies all `bar.sync` and `ret` instructions in PTX
2. Inserts a dispatch block that coordinates thread block retirement
3. Ensures all blocks reach synchronization points together
4. Prevents early-exit deadlocks

### PTX Modification Pattern

```ptx
// Original: bar.sync 0;
// Transformed:
mov.u32 %resume_idx, <block_id>;
setp.eq.u32 %sync_pred, 0, 0;  // has_sync = true
bra.uni $L__SYNC_BLOCK;

$L__POST_RET_OR_SYNC_BB_<id>:
// Continue after sync...
```

## Cubin Caching

To avoid repeated transformation overhead:

1. First kernel launch triggers transformation
2. Transformed PTX is compiled to fatbin via `nvcc`
3. Result is cached in `~/.cache/tally/transform/`
4. Subsequent launches use cached version
5. Cache persists across server restarts

---

# 内核转换设计

本文档描述 Tally 内核转换引擎的技术设计。

## 概述

Tally 在 PTX（并行线程执行）中间表示级别转换 CUDA 内核，无需修改应用源代码即可实现细粒度调度控制。

## 持久线程块（PTB）转换

### 概念

标准 CUDA 内核启动固定数量的线程块并执行到完成。PTB 转换重写内核，使用从共享计数器获取工作的持久线程块集合，实现：

- **受控占用率**：启动比原始更少的块，为其他工作负载留出 SM 资源
- **动态工作分配**：块通过原子递增全局索引自调度
- **抢占支持**：块可以检查撤退标志并提前退出

### PTB 变体

| 变体 | 描述 |
| --- | --- |
| 静态 PTB | 固定 blocks_per_sm，块运行到完成 |
| 动态 PTB | 块从全局计数器获取工作项 |
| 抢占式 PTB | 块定期检查撤退标志 |

## 内核切片

将网格划分为独立子网格，在切片边界实现抢占，适用于无法进行 PTB 转换的内核。

## 同步感知转换

当通过 PTB 启动比原始内核更少的线程块时，`__syncthreads()` 屏障可能导致死锁。同步感知转换识别所有同步点，插入协调逻辑确保所有块同时到达同步点。
