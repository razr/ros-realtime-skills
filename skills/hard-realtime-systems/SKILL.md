---
name: hard-realtime-systems
description: Enforces hard real-time systems programming rules for Linux PREEMPT_RT user-space applications in C++20 and Rust, including memory locking, SCHED_FIFO priority ceilings, priority inheritance, lock-free IPC, and IRQ isolation.
---

# Hard Real-Time Systems Programming (`PREEMPT_RT` C++20 & Rust)

Apply these mandatory invariants whenever writing, reviewing, or refactoring hard real-time control loops, device drivers, or low-latency robot software on Linux `PREEMPT_RT`.

## 1. Memory & Page-Fault Determinism
* **Lock All Process Memory at Startup:** Always call `mlockall(MCL_CURRENT | MCL_FUTURE)` (and pre-fault stack/heap pages) during initialization *before* entering the real-time loop.
* **Zero Dynamic Allocation on Hot Paths:** Never call `malloc`, `free`, `new`, `delete`, `Box::new`, `Vec::push`, or `String` formatting inside a `SCHED_FIFO` loop.
  * In C++20: Use pre-allocated static arenas, `std::array`, `std::span`, or deterministic TLSF allocators.
  * In Rust: Use `#![no_std]` or pre-allocated fixed buffers (`heapless::Vec`, static `[T; N]`), avoiding global allocator invocations on real-time threads.
* **Stop the Allocator Returning Memory:** In glibc programs call `mallopt(M_TRIM_THRESHOLD, -1)` and `mallopt(M_MMAP_MAX, 0)` at startup so `free()` never trims the heap or unmaps pages that the loop would fault back in.
* **Zero Disk or Blocking File I/O:** Never write logs, metrics, or files (`printf`, `std::cout`, `println!`, `write()`) directly from a real-time thread. Push binary records into a lock-free ring buffer drained by a low-priority housekeeping thread.

## 2. Scheduling, CPU Partitioning & Priority Ceilings
* **Strict Core Partitioning (`cgroups v2` / `sched_setaffinity`):** Pin real-time control threads exclusively to isolated P-cores (`isolcpus` / `cpuset.cpus.partition=isolated`). Never share cores with inference workloads, network daemons, or build/update tasks.
* **Explicit `SCHED_FIFO` Priority Hierarchy:** Assign deterministic priorities with clear separation—never set all threads to priority 99:
  * **P95:** Hardware Safety Supervisor / E-Stop watchdog.
  * **P92:** Deterministic Fieldbus I/O (`EtherCAT`, `CAN-FD`, `SPI`).
  * **P90:** Primary Control Loop (`1 kHz+` kinematics / impedance / motor loop).
  * **P60–P70:** High-rate deterministic sensor DMA ingestion (housekeeping cores only).
  * **`SCHED_OTHER` / `SCHED_IDLE`:** inference, telemetry, logging, and networking.
* **IRQ Affinity Steering:** Pin hardware interrupts (`/proc/irq/<irq>/smp_affinity`) for USB3, storage, and non-RT NICs strictly to non-RT housekeeping cores.

## 3. Synchronization & Lock-Free Concurrency
* **No Unbounded Locks on Real-Time Threads:** Never use standard `std::mutex` or `parking_lot::Mutex` across threads of different priorities (causes unbounded priority inversion).
* **Priority-Inheritance Mutexes or Lock-Free Queues:**
  * Prefer wait-free / lock-free Single-Producer Single-Consumer (**SPSC**) ring buffers (`boost::lockfree::spsc_queue`, `ringbuf`, or `Iceoryx2` shared memory) between RT and non-RT threads.
  * When a lock is unavoidable between RT threads, explicitly configure POSIX `PTHREAD_PRIO_INHERIT` (`rt_mutex` on `PREEMPT_RT`).
* **Multiple Producers:** Use a bounded lock-free MPMC queue (`boost::lockfree::queue` with a fixed size, `crossbeam::queue::ArrayQueue`) only when several producers or consumers are unavoidable; keep SPSC wherever the topology allows.
* **Bounded Execution Time:** Every loop iteration must have a provable Worst-Case Execution Time (**WCET**). Avoid unbounded `while` loops, recursive calls, or dynamic hash-map resizing on the hot path.

## 4. Periodic Timing
* **Sleep to Absolute Deadlines:** Drive the loop with `clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &next, nullptr)` and advance `next` by a fixed period each cycle. Relative sleeps accumulate drift and jitter.

## 5. Reference Initialization Pattern (C++20 / Rust)
```cpp
// C++20 Real-Time Thread Startup Checklist
#include <sys/mman.h>
#include <pthread.h>
#include <sched.h>
#include <cstdint>
#include <cstdlib>
#include <cstring>

if (mlockall(MCL_CURRENT | MCL_FUTURE) != 0) { std::abort(); }
// Pre-fault stack before entering SCHED_FIFO
volatile uint8_t stack_prefault[64 * 1024];
std::memset(const_cast<uint8_t*>(stack_prefault), 0, sizeof(stack_prefault));

struct sched_param param{};
param.sched_priority = 90;
if (pthread_setschedparam(pthread_self(), SCHED_FIFO, &param) != 0) { std::abort(); }

// Priority-inheritance mutex shared by RT threads of different priorities
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&mutex, &attr);
```

```rust
// Rust Real-Time Thread Startup Checklist (libc crate)
unsafe {
    if libc::mlockall(libc::MCL_CURRENT | libc::MCL_FUTURE) != 0 { std::process::abort(); }
    let param = libc::sched_param { sched_priority: 90 };
    if libc::pthread_setschedparam(libc::pthread_self(), libc::SCHED_FIFO, &param) != 0 {
        std::process::abort();
    }
}
// Pre-fault the stack with a fixed-size array touched before the loop starts.
```
