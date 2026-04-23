# supervised-multi-container-runtime

## Project Overview

This project implements a lightweight container runtime system that supports running and managing multiple containers under a supervised environment.
The runtime is designed to simulate fundamental features of container platforms like Docker, focusing on process control, resource management, and networking.

The implementation demonstrates core Operating Systems concepts such as:

* process isolation using namespaces
* inter-process communication (IPC)
* process lifecycle management
* synchronization using threads
* scheduling using nice values

---

## Features

- Multi-container creation and management

- Container lifecycle control (start, stop, monitor)

- Supervisor-based monitoring system

- Resource stress testing using CPU, memory, and I/O workloads

- Basic container execution environment using custom root filesystem

- Low-level interaction with Linux system calls

---

## Architecture

The project follows a modular structure with distinct components:

- Client/Workload Programs: Generate CPU, memory, and I/O stress

- Runtime Engine: Responsible for container execution

- Monitor Module: Observes and manages container behavior

- Root Filesystem: Provides isolated execution environment

---

## Environment

* Ubuntu 22.04 / 24.04 (Virtual Machine)
* Not supported on WSL
* Requires `sudo` for namespace operations

Install dependencies:

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

---

## Build Instructions

```bash
cd boilerplate
make
```

This builds:

* `engine` (user-space runtime)
* `monitor.ko` (kernel module)

---

## Running the Project

### 1. Start Supervisor

Terminal 1:

```bash
sudo ./engine supervisor ./rootfs-base
```

---

### 2. Start Containers

Terminal 2:

```bash
sudo ./engine start alpha ./rootfs-alpha /bin/sh
sudo ./engine start beta  ./rootfs-beta  /bin/sh
```

---

### 3. View Container Metadata

```bash
sudo ./engine ps
```

This shows:

* container ID
* PID
* state
* memory limits
* start time

---

## Logging Demonstration (Task 3)

To generate logs, run a workload inside the container.

```bash
cp cpu_hog rootfs-alpha/
```

Restart supervisor (important to clear stale socket):

```bash
rm -f /tmp/mini_runtime.sock
sudo ./engine supervisor ./rootfs-base
```

Run workload:

```bash
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog 15"
```

View logs:

```bash
cat logs/alpha.log
```

This demonstrates the bounded-buffer logging pipeline. 

---

## Stopping Containers

```bash
sudo ./engine stop alpha
```

---

## Kernel Monitor (Task 4)

Load kernel module:

```bash
sudo insmod monitor.ko
```

Verify:

```bash
ls -l /dev/container_monitor
```

---

### Memory Limit Test

```bash
cp memory_hog rootfs-alpha/
```

Terminal 1:

```bash
sudo ./engine supervisor ./rootfs-base
```

Terminal 3:

```bash
sudo dmesg -w
```

Terminal 2:

```bash
sudo ./engine start alpha ./rootfs-alpha "/memory_hog 5 500" --soft-mib 20 --hard-mib 35
```

Check metadata:

```bash
sudo ./engine ps
```

---

## Scheduling Experiment (Task 5)

Copy workloads:

```bash
cp cpu_hog rootfs-alpha/
cp cpu_hog rootfs-beta/
```

Start supervisor:

```bash
sudo ./engine supervisor ./rootfs-base
```

Run two containers with different priorities:

```bash
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog 30" --nice -5
sudo ./engine start beta  ./rootfs-beta  "/cpu_hog 30" --nice 5
```

Monitor CPU usage:

```bash
top
```

Compare completion times:

```bash
tail -3 logs/alpha.log
tail -3 logs/beta.log
```

Expected:

* lower nice value → faster execution
* higher nice value → slower execution

---

## Cleanup Verification (Task 6)

Stop containers:

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
```

Check for zombies:

```bash
ps aux | grep -E 'Z|defunct'
```

Check socket cleanup:

```bash
ls /tmp/mini_runtime.sock
```

Unload module:

```bash
sudo rmmod monitor
```

Check kernel logs:

```bash
dmesg | tail -3
```

---

## Key Concepts Demonstrated

* Supervisor-based container management
* Namespace isolation (PID, UTS, mount)
* UNIX domain socket IPC (CLI ↔ supervisor)
* Pipe-based logging (container → supervisor)
* Bounded buffer with producer-consumer threads
* Kernel-level memory monitoring (soft + hard limits)
* Linux scheduling using nice values

---

## Notes

* `/bin/sh` containers exit immediately (no logs)
* Workloads like `cpu_hog` are required to generate logs
* `rm -f /tmp/mini_runtime.sock` fixes stale socket issues
* `sudo` is required for most commands

---

## Conclusion
This project demonstrates the core principles behind container runtimes by building a simplified multi-container execution environment from scratch. It highlights how operating system concepts such as process creation, isolation, scheduling, and monitoring come together to manage concurrent workloads efficiently.

By implementing a supervisor-driven architecture along with workload generators, the project provides practical insight into how real-world systems ensure reliability and resource control. It also bridges the gap between user-space programs and kernel-level interactions, offering a deeper understanding of system-level programming.

Overall, this project serves as a solid foundation for exploring advanced container technologies and can be extended to incorporate features such as namespaces, cgroups, and networking to more closely resemble production-grade container runtimes.

---

## Engineering Analysis

### 1. Isolation Mechanisms
The runtime achieves isolation primarily through **Linux namespaces** and filesystem isolation techniques.

- **PID Namespace**: Ensures that processes inside a container have their own process ID space. Processes in one container cannot see or interfere with processes in another.
- **UTS Namespace**: Allows each container to have its own hostname and domain name, providing identity isolation.
- **Mount Namespace**: Gives each container its own view of the filesystem hierarchy, enabling independent mounts and preventing access to the host filesystem.

For filesystem isolation, techniques like **chroot** or **pivot_root** are used:
- `chroot` changes the apparent root directory for a process.
- `pivot_root` switches the root filesystem entirely, making the container filesystem independent from the host.

Despite these isolations, the **host kernel is shared across all containers**. This means:
- All containers use the same kernel
- Kernel-level resources (CPU scheduler, memory manager, device drivers) are shared
- Isolation is logical, not physical

---

### 2. Supervisor and Process Lifecycle
A long-running **supervisor process** is essential for managing container lifecycles.

- When a container is created, the supervisor uses **fork()** to create a child process and **exec()** to run the container workload.
- This establishes a **parent-child relationship**, where the supervisor tracks all container processes.
- The supervisor is responsible for:
  - Monitoring container state
  - Handling failures and restarts
  - Maintaining metadata (PID, status, resource usage)

When a container process exits, it becomes a **zombie process** until the parent reaps it using `wait()` or `waitpid()`. The supervisor ensures proper **process reaping** to avoid resource leaks.

Signals (e.g., SIGTERM, SIGKILL) are used to control container execution, allowing graceful or forced termination.

---

### 3. IPC, Threads, and Synchronization
The project uses multiple **Inter-Process Communication (IPC)** mechanisms such as:
- Pipes or sockets (for communication between components)
- Shared memory or buffers (for logging/monitoring)

A **bounded-buffer logging system** introduces concurrency challenges:
- Multiple producers (workers/containers) may write logs simultaneously
- A consumer (monitor/supervisor) reads logs

Possible race conditions include:
- Concurrent writes corrupting shared data
- Reading incomplete or inconsistent data

To handle this, synchronization mechanisms are used:
- **Mutexes** ensure mutual exclusion when accessing shared buffers
- **Condition variables / semaphores** manage coordination (e.g., buffer full/empty)
- These choices balance correctness and performance

---

### 4. Memory Management and Enforcement
**RSS (Resident Set Size)** measures the portion of a process’s memory that is currently loaded in physical RAM. However, it does not include:
- Swapped-out memory
- Shared memory fully attributed per process
- Kernel memory usage

The system distinguishes between:
- **Soft limits**: Advisory limits that trigger warnings or throttling
- **Hard limits**: Strict caps that cannot be exceeded

This distinction allows flexibility while still enforcing safety.

Memory enforcement must occur in **kernel space** because:
- Only the kernel has full visibility and control over physical memory
- User-space enforcement can be bypassed or is not authoritative
- The kernel ensures fair and secure allocation across processes

---

### 5. Scheduling Behavior
The observed behavior of workloads reflects how the **Linux scheduler** operates.

- CPU-intensive tasks (e.g., CPU hog) consume more CPU time but are balanced with others through fair scheduling
- I/O-bound tasks may get higher responsiveness due to blocking and waking behavior
- Memory-intensive workloads may be affected by paging and allocation delays

Linux scheduling aims to achieve:
- **Fairness**: Equal CPU time distribution among processes
- **Responsiveness**: Quick handling of interactive or I/O-bound tasks
- **Throughput**: Maximizing total work done over time

Experimental results show that the scheduler dynamically balances these goals, adjusting execution based on workload characteristics.
