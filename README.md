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

