# Kubernetes resources
| Resource type	| Description | Base unit |
| ------------- | ----------- | -------- |
| cpu | Compute processing | cpu (core) |
| memory | RAM | Bytes |
| ephemeral-storage | Local ephemeral storage | Bytes |
| hugepages-\<size\> | Huge pages (Linux only) | Bytes |
  
Clusters can also provide extended resources (resources with a custom name, typically exposed by device plugins).  
On Linux, container runtime typicaly uses kernel cgroups which applies and enforces limits/requests you define.

## CPU
Limits and requests are measured in cpu units. 1 CPU unit is equal tp 1 physical CPU core, or 1 virtual core.

### Requests
CPU requests define a relative scheduling weight, not a hard cap.

Kubernetes converts the requested CPU to cgroup CPU shares:
- cgroup v1: `cpu.shares`
- cgroup v2: `cpu.weight`

For cgroup v1, Kubernetes computes shares roughly as:

`cpu.shares = max(2, min(262144, request_mCPU * 1024 / 1000))`

So:
- `1000m` CPU -> `1024` shares
- `500m` CPU -> `512` shares
- `100m` CPU -> `102` shares

For cgroup v2, the equivalent property is `cpu.weight` in range `[1, 10000]`.

Historically, the v1 -> v2 conversion used the linear formula:

`cpu.weight = 1 + ((cpu.shares - 2) * 9999) / 262142`

Newer OCI/runtime stacks use a better mapping that keeps the important anchor points
`2 -> 1`, `1024 -> 100`, and `262144 -> 10000`:

`cpu.weight = ceil(10^(((log2(cpu.shares)^2 + 125*log2(cpu.shares)) / 612) - 7/34))`

[source - 04/2025](https://github.com/kubernetes/kubernetes/issues/131216)

Important: this conversion is runtime-version dependent. Older environments may still use the old linear mapping.

### Limits
CPU limits define a hard cap enforced by CFS bandwidth control.

- cgroup v1: `cpu.cfs_quota_us` + `cpu.cfs_period_us`
- cgroup v2: `cpu.max`

The default period is usually `100000` microseconds (`100ms`).

Quota is computed approximately as:

`quota_us = limit_mCPU * period_us / 1000`

Examples with the default `100000us` period:
- `1000m` CPU -> `100000us / 100000us`
- `500m` CPU -> `50000us / 100000us`
- `200m` CPU -> `20000us / 100000us`

This means a container can use up to that much CPU time during each `100ms` window before being throttled.

## Memory
Memory is specified in raw bytes or with quantity prefixes E, P, T, G, M, k or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki.  

### Requests
Memory requests are mainly a scheduling input.

They tell the scheduler how much memory the Pod should be guaranteed on a node.
Unlike CPU requests, memory requests are not a hard runtime limit by themselves.

At runtime:
- Pod QoS affects `oom_score_adj`, which influences which processes are more likely to be killed under node memory pressure.
- On cgroup v2, the runtime may also use memory requests to set `memory.low` or `memory.min`, depending on kubelet/runtime configuration and features such as MemoryQoS.

QoS classes and their typical `oom_score_adj` behavior:
- `BestEffort` -> `1000`
- `Guaranteed` -> `-997`
- `Burstable` -> somewhere in between, based on requested memory relative to node capacity

So memory requests help with:
- scheduling placement
- eviction / OOM priority
- optional memory protection on cgroup v2

They do not by themselves prevent a container from using more memory.

### Limits
Memory limits define the maximum memory a container or Pod cgroup is allowed to use.

- cgroup v1: `memory.limit_in_bytes`
- cgroup v2: `memory.max`

If the workload exceeds the memory limit, the kernel may trigger an OOM kill in that cgroup.

Unlike CPU, memory limits are not implemented as throttling. A container can sometimes go slightly over the limit temporarily, but if reclaim cannot bring usage back down, the kernel kills processes in that cgroup.

If no memory limit is set:
- the container may keep consuming memory as long as the node has it available
- under node-wide memory pressure, the kernel OOM killer and Kubernetes eviction logic decide what gets killed first
- Pods with worse QoS and higher memory usage are generally at higher risk