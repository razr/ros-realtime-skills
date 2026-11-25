---
name: ros2-realtime
description: Enforces deterministic ROS 2 and ros2_control patterns including StaticSingleThreadedExecutor, TLSF allocators, bounded QoS, zero-copy Iceoryx2/Zenoh transport, and safety-supervisor command clamping.
---

# Deterministic ROS 2 & `ros2_control` Engineering

Apply these rules whenever generating or modifying ROS 2 nodes, `ros2_control` hardware interfaces, executors, or sensor/AI pipelines for real-time robotics.

## 1. Executor & Callback Determinism
* **Use `StaticSingleThreadedExecutor` or `EventsExecutor`:** Never use `MultiThreadedExecutor` (or the single-threaded executor that `rclcpp::spin` creates by default) for hard real-time control nodes; use `rclcpp::executors::StaticSingleThreadedExecutor` (or a dedicated real-time `ros2_control` update thread) so callback entities are scanned statically without runtime heap allocations.
* **Custom Real-Time Allocators:** Configure `rclcpp` and middleware entities with a deterministic **TLSF** (`tlsf_cpp`) or bounded pool allocator (`rcl_allocator_t`) after `mlockall()`.
* **Use `realtime_tools` for Hand-Off:** Move data between non-real-time callbacks and the control loop with `realtime_tools::RealtimeBuffer` (non-blocking reads in the loop), and publish from the loop with `realtime_tools::RealtimePublisher`, which publishes from a separate non-real-time thread.
* **Preallocate Messages:** Allocate and size messages once before the loop; avoid `std::string` and unbounded sequences in real-time messages, and reuse the same message object each cycle.
* **Keep `ros2_control` `read()` / `update()` / `write()` Pure:** Inside `hardware_interface::SystemInterface::read()` and `write()`, perform only bounded register/fieldbus I/O and lock-free buffer reads/writes. Never call `RCLCPP_INFO`, `get_parameter()`, or blocking service clients inside the control cycle.

## 2. Deterministic QoS & Zero-Copy Transport
* **Explicit Bounded QoS Profiles:** Never rely on default QoS for real-time topics.
  * **High-Frequency Sensor / State Streams:** `ReliabilityPolicy::BestEffort`, `HistoryPolicy::KeepLast(1)`, `DurabilityPolicy::Volatile`.
  * **Safety / Mode / Trajectory Setpoints:** `ReliabilityPolicy::Reliable`, `HistoryPolicy::KeepLast(1)`, explicit `Deadline` and `Liveliness` policies so missed deadlines trigger deterministic fallback handlers.
* **Zero-Copy Intra-Host IPC (`Iceoryx2` / `rmw_zenoh` SHM):**
  * Never pass raw high-bandwidth camera images or dense point clouds through standard serialized DDS sockets.
  * Use **`Iceoryx2`** (`loan_uninit()`) or **`rmw_zenoh` shared memory** for Sensor -> AI zero-copy transfers.
  * **Hard Isolation Rule:** Hard real-time control nodes (`SCHED_FIFO`) must **never** subscribe to raw camera/LiDAR topics—only to compact, bounded geometric outputs (`< 1 KB` 6D pose, obstacle bounds, or velocity deltas).

## 3. Mixed-Criticality AI & Safety Supervision
* **Command Clamping & Watchdog Fallback:** Any trajectory or velocity command arriving from an AI/perception node must pass through a deterministic **Safety Supervisor** that enforces joint position/velocity/torque limits and falls back to a safe hold/stop if no valid update arrives within the `Deadline` window.
* **IT/OT Network Separation:** Isolate real-time fieldbus traffic (`EtherCAT`, `IEEE 1588 PTP` hardware timestamping) on a dedicated physical NIC, routing external telemetry asynchronously over `Zenoh` on housekeeping cores.
