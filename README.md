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
*(To be populated with annotated screenshots during development)*
1. **Multi-container supervision:** [Pending]
2. **Metadata tracking:** [Pending]
3. **Bounded-buffer logging:** [Pending]
4. **CLI and IPC:** [Pending]
5. **Soft-limit warning:** [Pending]
6. **Hard-limit enforcement:** [Pending]
7. **Scheduling experiment:** [Pending]
8. **Clean teardown:** [Pending]

## Engineering Analysis
*(To be populated in subsequent phases)*

### 1. Isolation Mechanisms
### 2. Supervisor Lifecycle
### 3. IPC & Synchronization
### 4. Memory Management (Kernel Module)
### 5. Scheduling Experiments

## Design Decisions and Tradeoffs
*(To be populated during development for each major subsystem)*

## Scheduler Experiment Results
*(To be populated in final phase)*
