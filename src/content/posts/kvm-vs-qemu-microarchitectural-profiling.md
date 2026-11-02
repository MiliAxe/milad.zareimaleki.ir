---
title: 'Microarchitectural Profiling: Hardware-Assisted KVM vs QEMU TCG Binary Translation'
description: 'Analyzing cache hierarchies, translation blocks, and SoftMMU overhead by comparing hardware virtualization against dynamic binary translation.'
pubDate: 2026-01-20
author: 'Milad Zarei-Maleki'
tags: ['virtualization', 'qemu', 'kvm', 'perf', 'microarchitecture', 'flamegraphs']
---

Virtualization strategies fundamentally govern how memory access and instruction scheduling manifest on the host silicon. In this study, we conducted microarchitectural profiling comparing:
1. **Hardware-Assisted Virtualization**: Linux KVM utilizing Intel VT-x hardware virtualization extensions.
2. **Dynamic Binary Translation**: QEMU TCG (Tiny Code Generator) performing cross-architecture translation (`x86_64` to `aarch64`).

## Workload Characterization

To expose different corners of the host memory hierarchy and CPU execution pipelines, we evaluated three distinct workload classes:

- **Pointer Chasing**: Highly non-sequential pointer traversal designed to defeat hardware prefetchers and isolate L1/L2/LLC cache miss penalties.
- **GEMM (General Matrix Multiply)**: Cache-blocking matrix multiplications stressing SIMD pipelines and memory throughput.
- **Multi-Threaded Sysbench**: Heavy lock contention, context switching, and I/O scheduler overhead.

Each test was pinned to isolated physical cores using `taskset -c ...` to eliminate thread migration jitter and CPU frequency scaling discrepancies.

```bash
# Pinning isolated vCPUs and collecting hardware performance counters
perf stat -e cycles,instructions,cache-references,cache-misses,branch-misses \
    taskset -c 2,3 qemu-system-x86_64 \
    -enable-kvm \
    -cpu host -smp 2 -m 4G \
    -kernel vmlinuz -initrd rootfs.cpio.gz
```

## Hardware Performance Counters & FlameGraphs

Using `perf record -g` and `perf mem`, we generated call-graph FlameGraphs to locate the execution hotspots:

- Under **KVM**, virtual CPU exits (`VM-Exit`) for non-root ring transitions accounted for the majority of virtualization overhead, but native memory accesses bypassed host CPU emulation entirely via Extended Page Tables (EPT / SLAT).
- Under **QEMU TCG**, host profile analysis revealed that up to 48% of CPU cycles were spent inside the **SoftMMU** emulation routines (`helper_le_ldul_mmu`) and Translation Block (TB) compilation/lookup tables.

Understanding these microarchitectural bottlenecks is critical when selecting emulation modes for testing embedded architectures and high-density virtualization setups.
