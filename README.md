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
3. **Bounded-buffer logging:** [Pending]
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
*(To be populated during Task 3)*

### 4. Memory Management (Kernel Module)
### 5. Scheduling Experiments

## Design Decisions and Tradeoffs
- **Namespace Selection:** Chose `clone()` flags for isolation. 
    - **Tradeoff:** Increases complexity in stack management and process synchronization compared to simple `fork()`.
    - **Justification:** Essential for meeting the security and isolation requirements of a true container runtime.

## Scheduler Experiment Results
*(To be populated in final phase)*
