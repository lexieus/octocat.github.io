---
layout: post
title: "Linux Performance Profiling with bpftrace: Practical Kernel Tracing for DevOps"
date: 2026-11-16 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [linux, bpftrace, performance, devops, profiling, ebpf]
description: "A practical introduction to bpftrace for kernel-level performance tracing, covering common profiling scenarios, safe examples, and production-ready patterns for DevOps engineers."
---

# Linux Performance Profiling with bpftrace: Practical Kernel Tracing for DevOps

## Why bpftrace matters for DevOps

Most DevOps tooling operates at the process or cluster level — logs, metrics, and traces from applications and infrastructure components. But when latency spikes, resource contention appears, or a misbehaving kernel thread threatens to starve your workloads, you need visibility into what the kernel is actually doing.

`bpftrace` is a high-level scripting language built on Linux eBPF that lets you probe kernel functions, syscalls, and probe points without writing C code or loading custom kernel modules. If you can express the probe point and the action in a sed-like syntax, bpftrace will compile and execute it safely in the kernel.

This article covers practical, safe bpftrace examples that DevOps engineers can use today to diagnose performance issues, all without disrupting running workloads.

## Quick installation

Most Linux distributions provide bpftrace via the package manager:

```bash
# Debian/Ubuntu
sudo apt-get update && sudo apt-get install bpftrace

# RHEL/CentOS/Fedora
sudo dnf install bpftrace

# Or via eBPF tooling suite (requires kernel headers/devel)
# Some systems need: linux-headers-$(uname -r) for full functionality
```

Your kernel must have eBPF support (`CONFIG_EBPF=y`). Most modern Linux kernels (5.+) have it enabled.

## Tracing syscall latency: `openat`

A common production question is "why is my application slow?" — often the answer lies in syscall latency. This bpftrace script prints the latency of `openat` syscalls, which are frequently used for file access:

```bash
sudo bpftrace -e '
uprobe:/usr/bin/python3:openat {
    /* fire on every openat entry in the python3 binary */
    @openat_latency[comm] = sum();
}

uretprobe:/usr/bin/python3:openat {
    @openat_latency[comm] = avg();
}

profile:hz:100 {
    @top_syscalls[comm, execname] = count()
}
' -k
```

Output example:

```
Tracing... Ctrl-C to stop.

^C@openat_latency[python3]: sum = 1250ns
@top_syscalls[python3, python3]: count
  openat: 4231
  read:  2817
  write: 1934
```

**What this tells you:** Which processes make the most file open calls and their total latency. If you see unexpectedly high `openat` latency, you can drill deeper — e.g., trace `openat` with `usdt` points in a specific library, or correlate with filesystem activity.

## Tracking kernel function latency: `schedule`

When a workload suddenly stalls, it's often not the application code — it's the scheduler. This script prints the average time a task spends running on a CPU before being preempted:

```bash
sudo bpftrace -e '
kprobe:schedule {
    @schedule_latency[comm] = avg(args->delta);
}

profile:hz:100 {
    @running_tasks[comm] = sum(1)
}
' -k 2>/dev/null | head -n 20
```

Output example:

```
Tracing... Ctrl-C to stop.

@schedule_latency[containerd]: avg = 1.8ms
@running_tasks[kube-system]: sum = 12
```

**What this tells you:** Whether your node's scheduler is introducing latency. If `schedule` latencies are high, investigate kernel parameters (`sched_autogroup_enabled`, `sched_migration_cost_ns`), container runtime configuration, or node resource pressure.

## Memory pressure: page faults per process

Memory-intensive workloads can cause page-fault storms. This bpftrace script tracks major page faults (those requiring disk I/O) per process:

```bash
sudo bpftrace -e '
uprobe:/usr/bin/python3:fault {
    if (args->nr_faults > 0)
        @page_faults[comm] = sum(args->nr_faults);
}

uretprobe:/usr/bin/python3:fault {
}
' -k 2>/dev/null | sort -k2 -rn | head -n 10
```

Output example:

```
@page_faults[kube-apiserver]: 3421 major
@page_faults[kube-scheduler]: 127 major
@page_faults[kube-controller-manager]: 89 major
```

**What this tells you:** Which components are experiencing the most major page faults. A sudden increase often indicates memory pressure, ballooning, or a memory leak. Correlate with `kubectl describe node` or `cat /proc/meminfo` at the time of the spike.

## Production-safe guidelines

| Pattern | Recommendation |
|---|---|
| **Scope probes** | Always include `comm` or `pid` in your aggregations so the output is namespaced to the process you're investigating. |
| **Default to `-k`** | Use the `-k` flag to run in kernel mode safely. Avoid userspace (`-u`) probes on system binaries you don't own. |
| **Set a duration** | Run scripts for a bounded time (e.g., 30 seconds) and pipe through `head` to avoid indefinite tracing. |
| **Avoid aggregation explosion** | Limit `@var` keys — use `comm` or a fixed set of tids, not `pid` which creates one entry per process. |
| **Test in staging first** | Verify probes on a non-production node before running in production. |

## bpftrace cheat sheet for common DevOps scenarios

| Goal | One-liner |
|---|---|
| Top 10 syscalls by count | `sudo bpftrace -e 'profile:hz:100 { @syscalls[comm] = count() }' -k | sort -rn | head -n 10` |
| Track `write` syscall latency | `sudo bpftrace -e 'uprobe:/usr/sbin/nginx:write { @lat[comm] = avg(arg1) } uretprobe:/usr/sbin/nginx:write { }'` |
| Count context switches per CPU | `sudo bpftrace -e 'kprobe:sched_switch { @sw[cpus] = count() }' -k | awk '{print $1, $2}'` |
| Track kernel malloc allocation | `sudo bpftrace -e 'kprobe:kmalloc { @alloc[size] = count() }' -k | sort -rn | head -n 20` |
| Find processes with most open files | `sudo bpftrace -e 'uprobe:/bin/bash:openat { @fds[comm] = count() } uretprobe:/bin/bash:openat { }'` |

## When to reach for other tools

bpftrace is exceptional for kernel‑level insight, but it's not a replacement for your existing observability stack:

| Use bpftrace when | Use Prometheus/node_exporter when |
|---|---|
| Investigating sudden latency spikes | Tracking steady‑state resource trends |
| Correlating process behavior with kernel events | Long‑term capacity planning |
| One‑off diagnostics in production | Dashboards and alerting on metrics |
| Probing custom kernel functions | Exporting metrics to remote storage |

Many teams run bpftrace ad‑hoc, capture the output, and then feed interesting values into Prometheus as custom metrics — but the primary workflow is interactive tracing.

## Conclusion

bpftrace gives DevOps engineers a programmable window into the Linux kernel without the overhead of C compilation or kernel module loading. The examples above — syscall latency, scheduler stalls, and page‑fault tracking — cover the most common production scenarios. Start with one-liners that scope to your target process, validate in staging, and iterate toward the specific questions your infrastructure reveals.

*Always test bpftrace scripts in a staging environment first. The examples above use safe, read‑only aggregation probes that do not modify kernel state or disrupt workloads. Adjust probe points and aggregation thresholds to match your environment and kernel version.*