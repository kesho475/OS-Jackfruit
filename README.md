# OS-Jackfruit: Custom Linux Container Runtime

**Team Members:**
* Shobhit Rachit Keshri (SRN: PES2UG24CS475 ; GitHub: kesho475)
  

## Build, Load, and Run Instructions

### Environment Setup
- **Host OS:** Ubuntu 22.04 (Bare Metal Boot)
- **Kernel Version:** 6.17+
- **Root Filesystem:** Alpine Linux Mini-Rootfs (v3.20.3)

**Initial Setup Completed:**
1. Bypassed the VM restriction in `environment-check.sh` to allow execution on bare metal Ubuntu.
2. Updated `monitor.c` (`timer_delete_sync`) to ensure compatibility with modern Linux kernels (6.15+).
3. Downloaded and extracted the Alpine mini-rootfs into the `rootfs-base/` directory to serve as the isolated filesystem template.

### Execution Guide

To reproduce the project demonstration, open two terminal windows (one for the long-running daemon, and one for the CLI client) and execute the following sequence.

#### 0. Pre-Run Clean
Ensure a pristine environment by clearing any stale processes, kernel modules, log files, or mounts.

**Terminal 2**
```bash
sudo killall sleep cpu_hog io_pulse mem_hog 2>/dev/null
sudo killall engine 2>/dev/null
sudo rmmod monitor 2>/dev/null
sudo umount ~/OS-Jackfruit/rootfs-*/proc 2>/dev/null
sudo rm -rf ~/OS-Jackfruit/logs/*
```

#### 1. Setup and Initialization
Compile the user-space binaries, load the kernel monitor, and start the supervisor daemon.

**Terminal 1 (Supervisor)**
```bash
cd ~/OS-Jackfruit/boilerplate
make
cd ~/OS-Jackfruit
sudo insmod boilerplate/monitor.ko
sudo ./boilerplate/engine supervisor ./rootfs-base
```
*(Leave Terminal 1 running in the background)*

#### 2. Multi-Container Supervision
Launch multiple isolated background containers and verify their tracking metadata.

**Terminal 2 (Client)**
```bash
cd ~/OS-Jackfruit
sudo ./boilerplate/engine start alpha ./rootfs-alpha "sleep 100"
sudo ./boilerplate/engine start beta ./rootfs-beta "sleep 100"
sudo ./boilerplate/engine ps
```

#### 3. Memory Enforcement
Demonstrate the kernel monitor's ability to enforce soft and hard Resident Set Size (RSS) limits.

**Terminal 2 (Client)**
```bash
sudo ./boilerplate/engine stop alpha
sudo ./boilerplate/engine start memtest ./rootfs-alpha "/mem_hog" --soft-mib 2 --hard-mib 10
sleep 15
sudo dmesg | tail -n 10
```

#### 4. Scheduling & Bounded-Buffer Logging
Run concurrent CPU and I/O workloads with adjusted priorities and verify the log-capture pipeline.

**Terminal 2 (Client)**
```bash
sudo ./boilerplate/engine stop memtest
sudo ./boilerplate/engine stop beta
sudo ./boilerplate/engine start hog ./rootfs-alpha "/cpu_hog" --nice 10
sudo ./boilerplate/engine start pulse ./rootfs-beta "/io_pulse"
sleep 5
sudo ./boilerplate/engine ps
cat logs/hog.log
cat logs/pulse.log
```

#### 5. Clean Teardown
Prove that all system resources, namespaces, and kernel allocations are cleanly released.

**Terminal 2 (Client)**
```bash
sudo ./boilerplate/engine stop hog
sudo ./boilerplate/engine stop pulse
```

**Terminal 1 (Supervisor)**
Press `Ctrl+C` to gracefully shut down the supervisor daemon.

**Terminal 2 (Client)**
```bash
sudo rmmod monitor
sudo dmesg | tail -n 5
ps aux | grep engine
sudo umount ~/OS-Jackfruit/rootfs-*/proc
```

## Demo Screenshots
1. **Multi-container supervision:** [Completed] - Shows multiple `sleep` processes running under the supervisor.
2. **Metadata tracking:** [Completed] - Output of the `ps` command showing tracked container metadata including PIDs and resource limits.
3. **Bounded-buffer logging:** [Completed] - Evidence of container output captured into log files via the concurrent pipeline.
4. **CLI and IPC:** [Completed] - Shows successful "start" commands issued via CLI to the supervisor daemon.
5. **Soft-limit warning:** [Completed] - `dmesg` output showing a soft-limit warning event when the `mem_hog` container crossed 2 MiB RSS.
6. **Hard-limit enforcement:** [Completed] - `dmesg` output showing the `mem_hog` container being killed via `SIGKILL` after exceeding its 10 MiB hard limit.
7. **Scheduling experiment:** [Completed] - Terminal output showing concurrent execution of `cpu_hog` (nice 10) and `io_pulse` (nice 0).
8. **Clean teardown:** [Completed] - Terminal output showing the successful `rmmod` command, `dmesg` unload confirmation, and clean `ps` process tree.

## Engineering Analysis

### 1. Isolation Mechanisms
- **Namespace Isolation:** The runtime utilizes the `clone()` system call with `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS` to provide each container its own process ID space, hostname, and mount points.
- **Filesystem Isolation:** Each container is confined to a dedicated writable copy of the Alpine rootfs using `chroot` and `chdir`. 
- **Kernel-level Sharing:** While namespaces isolate resource visibility, the containers still share the host's underlying Linux kernel and hardware resources.

### 2. Supervisor Lifecycle
- **Parent-Child Relationships:** The long-running supervisor acts as the reaper for container processes, ensuring that `SIGCHLD` signals are handled to prevent the accumulation of zombie processes.
- **Control Plane IPC:** A UNIX domain socket at `/tmp/mini_runtime.sock` allows the CLI client to send requests to the supervisor, enabling management of multiple concurrent container lifecycles.

### 3. IPC & Synchronization
- **Producer-Consumer Logging:** Implemented a thread-safe bounded buffer to decouple container output (producers) from file I/O (consumer).
- **Race Condition Prevention:** Without synchronization, multiple producer threads could overwrite the same buffer index simultaneously, or the consumer could attempt to read stale data.
- **Synchronization Choice:** - **`pthread_mutex_t`:** Used to ensure mutual exclusion when accessing shared buffer indices and the `count` variable.
    - **`pthread_cond_t`:** Used `not_full` and `not_empty` signals to prevent busy-waiting. This avoids deadlocks by putting threads to sleep when the buffer is full (producers) or empty (consumer).

### 4. Memory Management (Kernel Module)
- **RSS (Resident Set Size):** Measures the physical RAM currently held by the process. It is a critical metric for enforcement because it represents the actual pressure the container puts on the host's memory subsystem.
- **Kernel Enforcement:** Enforcement belongs in kernel space because it must be performed in an atomic, high-priority context (timer interrupt) that cannot be bypassed or delayed by user-space process scheduling.
- **Policy Differences:** - **Soft Limit:** Triggers a telemetry warning to signal that a container is exceeding its expected baseline.
    - **Hard Limit:** Triggers immediate termination via `SIGKILL` to prevent a rogue container from causing a system-wide Out-of-Memory (OOM) panic.

### 5. Scheduling Behavior
- **Scheduler Dynamics:** The Linux Completely Fair Scheduler (CFS) balances CPU time across processes. 
- **I/O-Bound vs CPU-Bound:** The `io_pulse` workload frequently yields the CPU while waiting for write operations to complete. The scheduler rewards this behavior with lower-latency wakeups to maintain responsiveness. Conversely, `cpu_hog` consumes its entire timeslice.
- **Priority Adjustment:** Applying `--nice 10` to `cpu_hog` lowered its scheduling weight, ensuring the CPU-bound task did not starve the I/O-bound process or the supervisor daemon.

## Design Decisions and Tradeoffs
- **Namespace Selection:** Chose `clone()` flags for isolation. 
    - **Tradeoff:** Increases complexity in stack management and process synchronization compared to simple `fork()`.
    - **Justification:** Essential for meeting the security and isolation requirements of a true container runtime.
- **Bounded Buffer Capacity:** Set a fixed capacity of 16 items.
    - **Tradeoff:** High-volume logging might block producers if the consumer is slow.
    - **Justification:** Prevents the supervisor from consuming excessive memory under heavy load.

## Scheduler Experiment Results
**Experiment Setup:**
- Workload A: `cpu_hog` (CPU-bound) running with `--nice 10` priority.
- Workload B: `io_pulse` (I/O-bound) running with default priority.

**Observations:**
Both containers executed concurrently. `io_pulse` immediately began iterating through its write loops, demonstrating rapid scheduler attention. `cpu_hog` processed calculations in the background, but its reduced priority ensured it gracefully yielded resources to the I/O operations and the supervisor's IPC handlers.
