---
name: realtime-verification-mcdc-ebpf
description: Enforces modern Linux eBPF observability (aya Rust, USDT probes) instead of LTTng and guides MC/DC test coverage compliance for safety-critical C++20 and Rust robotics code.
---

# Real-Time Verification: Native eBPF Tracing & MC/DC Coverage

Apply these verification rules when instrumenting real-time robotics code, building performance benchmarks, or writing unit tests for safety-critical modules.

## 1. Native Linux eBPF Observability (No Out-of-Tree LTTng)
* **Prefer Built-In Mainline eBPF over LTTng:** Do not require out-of-tree `lttng-modules` or `lttng-sessiond` on mainline `PREEMPT_RT` kernels.
* **Kernel Jitter & Isolation Auditing (`aya` / `libbpf` CO-RE):**
  * Attach eBPF programs to `sched:sched_switch`, `sched:sched_wakeup`, `irq:irq_handler_entry`, and `exceptions:page_fault_user` (x86) filtered by **`cgroup v2` ID** to prove zero page faults, zero IRQ leakage onto isolated P-cores, and adherence to `SCHED_FIFO` priority ceilings.
  * Drain events asynchronously from lock-free per-CPU ring buffers (`BPF_MAP_TYPE_RINGBUF`) on low-priority housekeeping cores.
* **Zero-Cost User-Space Probes (`USDT`):**
  * Use zero-cost static `USDT` probes (`usdt` crate in Rust, `sys/sdt.h` in C++) for hot-path `Iceoryx2` and `ros2_control` cycle timestamps.
  * For unmodified ROS 2 nodes, attach `uprobes` to the RMW implementation library (`librmw_implementation.so`) to capture topic publish/take rates and sequence drops without recompiling the nodes.

## 2. Automated MC/DC (Modified Condition/Decision Coverage)
* **Write Atomic, Measurable Boolean Decisions:** Avoid deeply nested ternary expressions or side-effecting function calls inside `if (...)` statements in safety supervisors and watchdogs so every condition can be independently toggled in unit tests.
* **C++20 MC/DC Compilation & Reporting:**
  * Compile safety-critical C++ targets with **GCC 14+ (`--coverage -fcondition-coverage`)** or **Clang 18+ (`-fprofile-instr-generate -fcoverage-mapping -fcoverage-mcdc`)**.
  * Export truth-table verification reports via `gcovr` or `llvm-cov show --show-mcdc`.
* **Rust MC/DC Compilation & Reporting:**
  * Run nightly LLVM MC/DC instrumentation (`RUSTFLAGS="-C instrument-coverage -Z coverage-options=mcdc"`) reported with `llvm-cov` in CI to verify masking MC/DC across all boolean branches.

## 3. Latency Measurement Gates
* **Measure Scheduling Latency with `cyclictest`:** Run it on the isolated cores with the same priority as the real-time loop, e.g. `cyclictest --mlockall --smp --priority=95 --duration=24h`. Record the maximum latency, not only the average, and fail CI when it exceeds the loop's deadline budget.
* **Attribute Outliers with `rtla`:** Use `rtla timerlat` (and `rtla osnoise`) to see whether a latency spike comes from IRQ handling, the scheduler, or other work on the core.

## 4. Mixed-Criticality Interference Benchmarking
* Never benchmark real-time loops solely on an idle system. Validate P99 and `max_lat` jitter under:
  1. **Memory/Cache Starvation:** `stress-ng --cache --vm --memcpy` on housekeeping cores.
  2. **Accelerator Contention:** Simultaneous GPU/NPU inference.
  3. **High-Bandwidth Sensor & Update Load:** Live camera DMA + software update / telemetry streaming.
