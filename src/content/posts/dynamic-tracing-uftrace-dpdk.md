---
title: 'Engineering Low-Overhead Dynamic Runtime Tracing with uftrace and DPDK'
description: 'How we extended uftrace for conditional function interception and profiled DPDK packet processing pipelines with bounded runtime overhead.'
pubDate: 2026-05-18
author: 'Milad Zarei-Maleki'
tags: ['systems', 'tracing', 'uftrace', 'dpdk', 'linux', 'performance']
---

Dynamic runtime tracing in high-throughput network stacks is notoriously challenging: conventional breakpoints or heavy instrumentation loops easily disrupt line-rate packet processing. In our research at the **Deep Observability Lab (IUST)**, supervised by Dr. Seyed Vahid Azhari, we investigated function interception and low-overhead conditional tracing in kernel and userspace environments using [`uftrace`](https://github.com/namhyung/uftrace).

## The Overhead Problem in High-Rate Packet Pipelines

When analyzing high-throughput networking frameworks like **DPDK** (Data Plane Development Kit), executing millions of packets per second per core means that an instrumentation penalty of even a few hundred nanoseconds per packet causes queue overflow and packet drops.

To address this:
1. We utilized compiler profiling hooks (`-pg` / `-mfentry`) inserted by GCC/Clang at function entry points.
2. We leveraged **DWARF-aware runtime struct resolution** to parse complex argument structs directly from memory without recompiling packet handlers with invasive probes.
3. We implemented dynamic predicate filters that evaluate execution conditions (e.g., packet header fields, call graph depth, thread IDs) before deciding whether to log a record to trace memory.

```c
/* Evaluating conditional predicates at runtime with bounded overhead */
int evaluate_packet_predicate(struct rte_mbuf *mbuf, const struct trace_filter *filter) {
    if (__builtin_expect(!mbuf, 0))
        return 0;

    struct rte_ether_hdr *eth = rte_pktmbuf_mtod(mbuf, struct rte_ether_hdr *);
    if (eth->ether_type == rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4)) {
        struct rte_ipv4_hdr *ip = (struct rte_ipv4_hdr *)(eth + 1);
        return (ip->next_proto_id == filter->target_proto);
    }
    return 0;
}
```

## Instrumenting `dpdk-testpmd`

We instrumented critical L3/L4 processing routines inside `dpdk-testpmd`, specifically focusing on `swap_ipv4` and `rte_ipv4_hdr_len`. By constructing headless benchmarking harnesses utilizing:
- 2MB/1GB Hugepages configured via `hugetlbfs`
- Virtual TAP interfaces (`ip tuntap add mode tap ...`)
- Isolated Linux network namespaces (`netns`)

We were able to verify bounded trace overhead while extracting cycle-accurate call trees under high burst rates.

## Upstream Contributions to uftrace

As part of this work, we contributed several fixes and test cases back to official upstream `uftrace`:

1. **Sub-64-bit Signed Argument Normalization**: In `uftrace_eval_cond()`, sub-64-bit signed integers (e.g. `int16_t`, `int8_t`) passed in 64-bit registers were not correctly sign-extended during condition evaluations. We normalized the bit representations across supported architectures.
2. **Negative Integer Filter Test Suite (`t306_arg_cond3`)**: Authored comprehensive automated test cases validating signed comparisons (`<`, `<=`, `>`, `>=`) against negative constants in trace arguments.
3. **Android & POSIX Shared Memory Leak Resolution**: Fixed resource cleanup paths for shared memory segments allocated during multi-process tracing runs.

Observability must not compromise performance; conditional execution filters and hardware compiler hooks provide the exact primitives needed to achieve this balance.
