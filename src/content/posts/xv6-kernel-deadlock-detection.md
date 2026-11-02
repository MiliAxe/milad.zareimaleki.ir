---
title: 'In-Kernel Dynamic Deadlock Detection Using Resource Allocation Graphs in xv6'
description: 'Extending the xv6 teaching kernel with runtime cycle detection in an active Resource Allocation Graph to prevent circular waits.'
pubDate: 2026-03-12
author: 'Milad Zarei-Maleki'
tags: ['os', 'xv6', 'c', 'kernel', 'deadlock', 'concurrency']
---

Deadlocks in monolithic operating system kernels are catastrophic because an unhandled circular dependency freezes system execution. In this project, we designed and implemented an active, in-kernel deadlock detection and prevention engine within the **xv6** operating system kernel (x86 / QEMU).

## Active Resource Allocation Graph (RAG)

Traditional deadlock recovery often relies on timeout heuristics or userspace supervisors. We embedded an explicit **Resource Allocation Graph (RAG)** directly into dedicated kernel memory pages:

- **Nodes**: Threads (processes) and Resources (locks, synchronization primitives, shared device handles).
- **Claim Edges**: A thread announces an intention to acquire a resource.
- **Assignment Edges**: Directed edge from resource $R_j \to T_i$, indicating the resource is held by thread $i$.
- **Request Edges**: Directed edge from thread $T_i \to R_j$, indicating thread $i$ is waiting for resource $j$.

```c
struct rag_node {
    int id;
    int is_resource;
    int holding_threads[MAX_THREADS];
    int waiting_threads[MAX_THREADS];
};

struct rag_table {
    struct spinlock lock;
    struct rag_node nodes[MAX_RAG_NODES];
    int edge_matrix[MAX_RAG_NODES][MAX_RAG_NODES];
};
```

## System Calls for Explicit Resource Tracking

To bridge userspace threads with the kernel detection engine, we introduced custom system calls:

- `sys_requestresource(int res_id)`: Registers a request edge from the current thread to the resource. Before placing the thread into a sleep state on the resource lock channel, the kernel invokes cycle detection.
- `sys_releaseresource(int res_id)`: Removes the assignment edge and transfers ownership to the next runnable thread in the wait-queue.
- `sys_readresource` / `sys_writeresource`: Granular shared vs exclusive access semantics.

## Cycle Detection with Stack State Tracking

Whenever a request edge is tentatively added to the graph, a Depth-First Search (DFS) traversal executes:

```c
// DFS cycle detection routine
static int check_circular_dependency(int start_node, int current_node, int visited[], int stack[]) {
    visited[current_node] = 1;
    stack[current_node] = 1;

    for (int next = 0; next < MAX_RAG_NODES; next++) {
        if (rag.edge_matrix[current_node][next]) {
            if (!visited[next] && check_circular_dependency(start_node, next, visited, stack))
                return 1;
            else if (stack[next])
                return 1; // Cycle detected!
        }
    }

    stack[current_node] = 0;
    return 0;
}
```

If a cycle is detected, the `requestresource` system call immediately aborts the transition, rolls back the tentative edge, and returns an error code (`EDEADLK`) to userspace, preemptively neutralizing the circular wait condition before any kernel thread sleeps.
