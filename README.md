# Pintos - Operating System Kernel Development

A production-grade implementation of core operating system subsystems for the 80x86 architecture. This project involves building out the threading, user program execution, virtual memory, and file system layers of the Stanford Pintos instructional operating system.

> **Note**: This repository is private to maintain academic integrity for the UT Austin MSCS Advanced Operating Systems course. This README serves as a portfolio showcase of the architecture and implementation details.

## Contributors

This project was implemented by:

* **Will Sullivan** ([@wsulliv8](https://github.com/wsulliv8))
* **Dylan Pina** ([@DylanPina](https://github.com/DylanPina))

## 🎯 Overview

Pintos is a simple but complete operating system framework for the 80x86 architecture. The base code provides minimal functionality, and this project fully implements the core subsystems required of a modern operating system. The system guarantees process isolation, efficient memory management, and thread-safe file operations in a highly concurrent environment.

### Key Features

* **Advanced Thread Scheduling**: Priority scheduling with priority donation (to prevent priority inversion) and a Multi-Level Feedback Queue Scheduler (MLFQS).
* **Process Management**: Safe execution of user programs with hardware-enforced memory protection, argument passing, and isolated system calls.
* **Virtual Memory**: Demand paging, lazy loading, stack growth, memory-mapped files (mmap), and eviction handling using a page replacement algorithm.
* **Robust File System**: An extensible file system with a buffer cache, indexed files, and hierarchical subdirectory support.
* **Concurrency & Synchronization**: Rock-solid synchronization primitives (semaphores, locks, condition variables) preventing race conditions across the kernel.

## 🏗️ Architecture

```text
┌────────────────────────────────────────────────────────┐
│                   User Programs                        │
│   (ls, cat, echo, custom test binaries, shell)         │
└──────────────────────────┬─────────────────────────────┘
                           │ System Calls (Interrupt 0x30)
                           ▼
┌────────────────────────────────────────────────────────┐
│                   Pintos Kernel                        │
│                                                        │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────┐  │
│  │   Thread   │  │   Virtual  │  │   File System    │  │
│  │ Scheduling │  │   Memory   │  │  (Buffer Cache)  │  │
│  └────────────┘  └────────────┘  └──────────────────┘  │
│                                                        │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────┐  │
│  │    Syscall │  │  Page      │  │   Block Device   │  │
│  │    Handler │  │  Fault     │  │   Interface      │  │
│  │            │  │  Handler   │  │                  │  │
│  └────────────┘  └────────────┘  └──────────────────┘  │
└──────────────────────────┬─────────────────────────────┘
                           │ Hardware Instructions
                           ▼
┌────────────────────────────────────────────────────────┐
│                   80x86 Hardware                       │
│     (CPU, Memory, Registers, Interrupt Controller,     │
│                 IDE Disk, Keyboard)                    │
└────────────────────────────────────────────────────────┘

```

The system is separated by a strict boundary between user space (Ring 3) and kernel space (Ring 0). User programs must trap into the kernel via system calls to request resources, and the kernel meticulously validates all user pointers to prevent malicious or accidental system crashes.

## 📋 Core Components

### 1. Threads and Scheduling

* **Alarm Clock**: Replaced busy-waiting with an efficient sleep queue, waking threads only when their sleep timer expires.
* **Priority Scheduling**: Implemented strict priority execution.
* **Priority Donation**: Solved the priority inversion problem by allowing low-priority threads holding locks to temporarily inherit the priority of high-priority threads waiting on those locks. Handles multiple and nested donations.
* **MLFQS**: Implemented a scheduler that dynamically adjusts thread priorities based on recent CPU usage to balance CPU-bound and I/O-bound tasks.

### 2. User Programs

* **Argument Passing**: Properly parsed and pushed command-line arguments onto the user stack before process execution.
* **Memory Protection**: Implemented strict validation of user pointers. Any attempt to read or write unmapped or kernel memory cleanly terminates the offending process.
* **System Call Interface**: Handled hardware interrupts to route user system calls to the appropriate kernel functions safely.
* **Process Synchronization**: Implemented wait and exit semantics ensuring parent processes can correctly wait on child execution states and retrieve exit codes.

### 3. Virtual Memory

* **Supplemental Page Table**: Tracked the location of pages (memory, swap, or file) to handle page faults efficiently.
* **Demand Paging**: Loaded pages lazily only when accessed by the user program, saving physical memory.
* **Frame Table & Eviction**: Managed physical memory frames. Implemented a clock/LRU-approximation algorithm to evict pages to swap space when memory is full.
* **Stack Growth**: Automatically allocated new pages when user programs exceeded their initial stack space.

### 4. File System

* **Buffer Cache**: Implemented an LRU cache for disk blocks, dramatically reducing I/O latency for frequent reads/writes. Read-ahead and write-behind optimizations included.
* **Extensible Files**: Replaced the contiguous allocation model with an indexed structure (direct, indirect, and doubly-indirect blocks), allowing files to grow dynamically up to the 8MB disk limit.
* **Subdirectories**: Expanded the flat root directory into a hierarchical tree, implementing path resolution for relative and absolute paths.

## 🔐 Safety & Synchronization Guarantees

Operating systems are inherently concurrent. This implementation guarantees:

* **Thread Safety**: Global kernel data structures are protected by fine-grained locking and interrupts are selectively disabled only during critical context switches.
* **Process Isolation**: A faulty or malicious user process cannot crash the kernel or access data belonging to another process.
* **Deadlock Avoidance**: Locks are acquired and released in a strict, predictable order.
* **Data Integrity**: File system operations use synchronization to ensure atomic writes and prevent disk corruption during concurrent accesses.

## 🧪 Testing and Debugging

The codebase passes the strict, automated testing suite provided by Stanford/UT Austin, executing inside the QEMU and Bochs emulators.

* **100+ Automated Tests**: Covering edge cases in page faulting, priority donation chains, memory leaks, and concurrent file system access.
* **Debug Tooling**: Extensive use of kernel panics, assertions (ASSERT), and custom backtrace mapping to debug x86 register states during page faults.

## 🛠️ Technology Stack

* **Language**: C99
* **Architecture**: 32-bit 80x86 (x86)
* **Assembly**: Inline x86 Assembly for context switching and hardware interrupt manipulation.
* **Build System**: GNU Make
* **Environment**: QEMU / Bochs PC emulators

## 📝 API Design / System Calls

The kernel exposes the following system calls to user programs:

### Process Control

* `halt()` - Shuts down the operating system.
* `exit(status)` - Terminates the current user program.
* `exec(cmd_line)` - Starts a new process.
* `wait(pid)` - Waits for a child process to exit.

### File System Operations

* `create(file, initial_size)` - Creates a new file.
* `remove(file)` - Deletes a file.
* `open(file)` - Opens a file and returns an FD.
* `filesize(fd)` - Returns the size of a file.
* `read(fd, buffer, size)` - Reads from a file.
* `write(fd, buffer, size)` - Writes to a file (synchronized for concurrency).
* `seek(fd, position)` - Changes the next byte to be read or written.
* `tell(fd)` - Returns the position in a file.
* `close(fd)` - Closes a file descriptor.

## 🙏 Acknowledgments

* **Stanford University**: Ben Pfaff and the original creators of the Pintos project.
* **UT Austin**: The MSCS Advanced Operating Systems course staff.
* **OSTEP**: *Operating Systems: Three Easy Pieces* by Remzi H. Arpaci-Dusseau and Andrea C. Arpaci-Dusseau for theoretical foundations.
