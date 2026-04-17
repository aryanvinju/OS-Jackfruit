# Multi-Container Runtime

## 1. Team Information

| Name | SRN |
|------|-----|
| ARYAN VINJU | PES2UG24AM056 |
| ADITYA SHETTY | PES2UG24AM018 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Prepare the Alpine root filesystem

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

### Build

```bash
make
```

Builds: `engine`, `monitor.ko`, `memory_hog`, `cpu_hog`, `io_pulse`.

### Copy workloads into rootfs

```bash
cp memory_hog cpu_hog io_pulse ./rootfs-alpha/
cp memory_hog cpu_hog io_pulse ./rootfs-beta/
```

### Load the kernel module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
```

### Terminal 1 — Start supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

### Terminal 2 — CLI commands

```bash
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog" --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  "/cpu_hog" --soft-mib 48 --hard-mib 80 --nice 10
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Memory limit test

```bash
sudo ./engine start alpha ./rootfs-alpha "/memory_hog" --soft-mib 10 --hard-mib 20
sudo ./engine start beta  ./rootfs-beta  "/memory_hog" --soft-mib 10 --hard-mib 20
sleep 8
sudo dmesg | grep container_monitor | tail -20
sudo ./engine ps
```

### Scheduling experiment

```bash
sudo rm -f logs/alpha.log logs/beta.log
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog" --nice 0  --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  "/cpu_hog" --nice 10 --soft-mib 48 --hard-mib 80
sleep 20
sudo ./engine stop alpha && sudo ./engine stop beta
echo "=== ALPHA ===" && sudo ./engine logs alpha | tail -5
echo "=== BETA  ===" && sudo ./engine logs beta  | tail -5
```

### Teardown

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
# Ctrl+C supervisor in terminal 1
ps aux | grep defunct
sudo rmmod monitor
sudo dmesg | tail -3
make clean
```
---
## Output Screenshots
Screenshot 1 — Multi-container supervision
<img width="975" height="169" alt="image" src="https://github.com/user-attachments/assets/3983544d-3fc8-47a7-8de3-bda22ff471e7" />

Screenshot 2 — Metadata tracking
<img width="975" height="493" alt="image" src="https://github.com/user-attachments/assets/21e6f4a2-83dd-4b38-b6d0-4b8b549a20e6" />

Screenshot 3 — Bounded-buffer logging
<img width="975" height="304" alt="image" src="https://github.com/user-attachments/assets/95be4cca-f67d-4c5e-9517-3b4a66fcc9f2" />

Screenshot 4 — CLI and IPC
<img width="975" height="275" alt="image" src="https://github.com/user-attachments/assets/e693f4fd-a244-44e1-bbc8-f25998f831aa" />


Screenshot 7 — Scheduling experiment
<img width="975" height="379" alt="image" src="https://github.com/user-attachments/assets/ed2f48f4-9388-45ff-81a8-070cb1269536" />


## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Linux namespaces are the kernel primitive that makes container isolation possible. Our runtime uses three:

**PID namespace** (`CLONE_NEWPID`): The child sees itself as PID 1 in its own PID tree and cannot signal or observe processes outside it. The host kernel assigns each container process a second namespace-local PID. This is why `ps` inside the container shows only its own processes.

**UTS namespace** (`CLONE_NEWUTS`): Each container gets its own hostname. We call `sethostname(container_id)` inside the child so `hostname` returns the container ID. The kernel still owns the network stack — only the name is virtualized.

**Mount namespace** (`CLONE_NEWNS`): The container gets its own filesystem tree view. Combined with `chroot()`, it sees only its assigned `rootfs` as `/`. We mount `/proc` inside so `ps` and `top` work. The host kernel still owns the underlying block devices — `chroot` restricts the view but does not provide true storage isolation.

What the host kernel still shares: the kernel itself, all system calls, the network stack (no `CLONE_NEWNET`), device files, and physical memory. Our kernel monitor enforces memory limits since containers can otherwise exhaust host RAM.

`pivot_root` would be more thorough than `chroot` — it fully replaces the root mount point preventing `..` traversal escapes. We use `chroot` for simplicity.

### 4.2 Supervisor and Process Lifecycle

A long-running parent supervisor is necessary because only the direct parent can `wait()` for a child. If the supervisor exited after spawning, containers would be orphaned by `init` and exit status metadata lost.

1. **Creation**: `clone()` with namespace flags creates the container child. Preferred over `fork()` for fine-grained resource sharing control.
2. **Parent-child relationship**: The supervisor holds each container's host PID for signalling, ioctl registration, and reaping.
3. **Reaping**: `SIGCHLD` handler with `SA_NOCLDSTOP` calls `waitpid(-1, WNOHANG)` in a loop. Without this, exited containers become zombies indefinitely holding process table slots.
4. **Metadata synchronization**: `container_record_t` linked list protected by `pthread_mutex_t metadata_lock` — both the `SIGCHLD` handler and CLI socket handler access it concurrently.
5. **Orderly shutdown**: `SIGTERM` sets `should_stop`, sends `SIGTERM` to all containers, waits 1s, `SIGKILL`s stragglers, joins logger thread, frees all heap.

### 4.3 IPC, Threads, and Synchronization

**Path A — Logging (pipes)**:
Container stdout/stderr connected via `pipe()`. Write end given to child via `dup2()`. Supervisor's log-reader thread reads from the read end and pushes chunks into the bounded buffer.

Bounded buffer synchronization:
- `pthread_mutex_t mutex`: prevents concurrent writes to head/tail/count. Without it, two producers could compute the same insertion index and overwrite each other.
- `pthread_cond_t not_empty`: consumer blocks when empty — no busy-spin.
- `pthread_cond_t not_full`: producers block when full — no silent data overwrite.

Consumer drains all remaining items after `shutting_down` is set — no log lines dropped on shutdown.

**Path B — Control plane (UNIX domain socket)**:
CLI connects to `/tmp/mini_runtime.sock`, sends `control_request_t`, receives streamed `control_response_t`. UNIX sockets chosen over FIFOs: bidirectional, `connect()`/`accept()` semantics, kernel-buffered without polling.

`stop_requested` flag set under `metadata_lock` before `SIGTERM` — allows `SIGCHLD` handler to atomically classify termination as manual stop vs. kernel-monitor kill.

### 4.4 Memory Management and Enforcement

**RSS measures**: Physical RAM pages currently mapped and faulted in — code, stack, heap, memory-mapped files that are resident.

**RSS does not measure**: Virtual memory mapped but not touched, swapped-out pages, kernel memory used on the process's behalf.

**Why two tiers**: Soft limit warns without stopping — gives the application a chance to respond. Hard limit kills unconditionally. This mirrors `RLIMIT_AS` soft/hard in `setrlimit()` and gives operators a configurable warning window.

**Why kernel space**: User-space polling of `/proc/<pid>/status` has TOCTOU races — by the time it reads RSS and issues kill, the process may have allocated more. The kernel timer callback has direct `mm_struct` access via `get_mm_rss()`, sends signals atomically with `send_sig()`, and cannot be killed by the monitored process.

### 4.5 Scheduling Behavior

CFS assigns each process a `vruntime` and always schedules the runnable process with the smallest `vruntime`. Nice values map to weights: nice=0→1024, nice=10→110, nice=19→35. CFS advances `vruntime` inversely proportional to weight — lower-weight processes accumulate `vruntime` faster and are scheduled less under CPU contention.

Our experiment showed `cpu_hog`'s `sleep(1)` pattern makes it sleep-dominated. Both containers advanced at the same rate regardless of nice value, confirming CFS's no-starvation guarantee while revealing the scheduling boundary condition: nice priority only operates on simultaneously runnable processes. Under true CPU saturation with a pure CPU-bound workload, nice=0 receives ~29× more CPU than nice=19 per the CFS weight table.

---

## 5. Design Decisions and Tradeoffs

| Subsystem | Choice | Tradeoff | Justification |
|-----------|--------|----------|---------------|
| Namespace isolation | `CLONE_NEWPID\|NEWUTS\|NEWNS`, no network namespace | Containers share host network stack | Network isolation requires veth/bridge setup out of scope; three namespaces fully demonstrate required isolation |
| Supervisor architecture | Single-threaded `select()` event loop | `CMD_RUN` blocks event loop during container wait | Simple, correct for project scale; blocking `run` matches spec requirement |
| IPC/logging | UNIX socket (control) + pipes (logging) + bounded buffer | Fixed buffer capacity may stall producers if consumer falls behind | Clean separation of concerns; decouples production from disk write speed |
| Kernel monitor | `spinlock` on entry list | Busy-waits; cannot sleep | Timer callback runs in softirq context where sleeping is prohibited — only correct choice |
| Scheduling experiments | `nice` via `--nice` flag + `nice()` syscall | Affects whole process tree; coarser than cgroup CPU shares | Simplest portable CFS weight interface; no cgroup infrastructure needed |

---

## 6. Scheduler Experiment Results

### Setup

| Container | Nice | CFS Weight | Expected CPU share (under contention) |
|-----------|------|-----------|---------------------------------------|
| alpha | 0 | 1024 | 1024/(1024+110) ≈ **90%** |
| beta | 10 | 110 | 110/(1024+110) ≈ **10%** |

### Raw Data

| elapsed (s) | alpha (nice=0) | beta (nice=10) |
|-------------|----------------|----------------|
| 1 | 7,995,303,363,978,707,087 | 14,211,033,025,340,614,531 |
| 2 | 2,042,901,725,829,384,323 | 16,902,921,991,183,175,343 |
| 3 | 5,649,328,118,814,881,648 | 19,719,482,046,531,964,733 |
| 5 | 9,498,136,114,780,567,093 | 16,987,805,286,090,671,302 |
| 9 | 181,350,287,107,098,816,000 | 149,098,526,440,063,023,722 |

### Analysis

Both containers advanced one elapsed-second per wall-clock second regardless of nice value.

**Root cause**: `cpu_hog` calls `sleep(1)` between each print. This removes it from the CFS run queue for 1 second per iteration — making it sleep-dominated rather than CPU-bound. CFS nice-value weights only redistribute CPU time among simultaneously runnable processes. A sleeping process's weight is irrelevant during sleep.

**What the results confirm**:
1. **No starvation**: Beta (nice=10) made steady progress every second — CFS guarantees forward progress for all runnable processes.
2. **Correct isolation**: Both containers ran fully independently in their own namespaces.
3. **Scheduling boundary**: Priority only matters under real CPU contention between runnable processes.

**Theoretical result with pure CPU-bound workload (no sleep)**:

| nice | weight | CPU share |
|------|--------|-----------|
| 0 | 1024 | ≈ 90% |
| 19 | 35 | ≈ 3.3% |

Alpha would complete **~29× more iterations** than beta in the same wall-clock window.

**Conclusion**: The experiment demonstrates container isolation, concurrent workload management, and the scheduling boundary condition — nice priority is a mechanism for arbitrating CPU contention, not for throttling voluntarily-yielding processes. Background batch jobs (high nice) are deprioritized only when foreground work actually needs the CPU, and are otherwise free to use available idle cycles.
