# OS-Jackfruit: Custom Linux Container Runtime

**Team Members:**
* Shobhit Rachit Keshri (GitHub: kesho475)

## Build, Load, and Run Instructions

### Environment Setup
- **Host OS:** Ubuntu 24.04 (Bare Metal Boot)
- **Kernel Version:** 6.17+
- **Root Filesystem:** Alpine Linux Mini-Rootfs (v3.20.3)

**Initial Setup Completed:**
1. Bypassed the VM restriction in `environment-check.sh` to allow execution on bare metal Ubuntu.
2. Updated `monitor.c` (`timer_delete_sync`) to ensure compatibility with modern Linux kernels (6.15+).
3. Downloaded and extracted the Alpine mini-rootfs into the `rootfs-base/` directory to serve as the isolated filesystem template.

*(Further build and run commands will be added as features are implemented.)*

## Demo Screenshots
1. **Multi-container supervision:** [Completed] - Shows multiple `sleep` processes running under the supervisor.
2. **Metadata tracking:** [Pending]
3. **Bounded-buffer logging:** [Completed] - Evidence of container output captured into log files via the concurrent pipeline.
4. **CLI and IPC:** [Completed] - Shows successful "start" commands issued via CLI to the supervisor daemon.
5. **Soft-limit warning:** [Pending]
6. **Hard-limit enforcement:** [Pending]
7. **Scheduling experiment:** [Pending]
8. **Clean teardown:** [Pending]

## Engineering Analysis

### 1. Isolation Mechanisms
- **Namespace Isolation:** The runtime utilizes the `clone()` system call with `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS` to provide each container its own process ID space, hostname, and mount points.
- **Filesystem Isolation:** Each container is confined to a dedicated writable copy of the Alpine rootfs using `chroot` and `chdir`. 
- **Kernel-level Sharing:** While namespaces isolate resource visibility, the containers still share the host's underlying Linux kernel and hardware resources.

### 2. Supervisor Lifecycle
- **Parent-Child Relationships:** The long-running supervisor act as the reaper for container processes, ensuring that `SIGCHLD` signals are handled to prevent the accumulation of zombie processes.
- **Control Plane IPC:** A UNIX domain socket at `/tmp/mini_runtime.sock` allows the CLI client to send requests to the supervisor, enabling management of multiple concurrent container lifecycles.

### 3. IPC & Synchronization
- **Producer-Consumer Logging:** Implemented a thread-safe bounded buffer to decouple container output (producers) from file I/O (consumer).
- **Race Condition Prevention:** Without synchronization, multiple producer threads could overwrite the same buffer index simultaneously, or the consumer could attempt to read stale data.
- **Synchronization Choice:** - **`pthread_mutex_t`:** Used to ensure mutual exclusion when accessing shared buffer indices and the `count` variable.
    - **`pthread_cond_t`:** Used `not_full` and `not_empty` signals to prevent busy-waiting. This avoids deadlocks by putting threads to sleep when the buffer is full (producers) or empty (consumer).

### 4. Memory Management (Kernel Module)
### 5. Scheduling Experiments

## Design Decisions and Tradeoffs
- **Namespace Selection:** Chose `clone()` flags for isolation. 
    - **Tradeoff:** Increases complexity in stack management and process synchronization compared to simple `fork()`.
    - **Justification:** Essential for meeting the security and isolation requirements of a true container runtime.
- **Bounded Buffer Capacity:** Set a fixed capacity of 16 items.
    - **Tradeoff:** High-volume logging might block producers if the consumer is slow.
    - **Justification:** Prevents the supervisor from consuming excessive memory under heavy load.

## Scheduler Experiment Results
*(To be populated in final phase)*
