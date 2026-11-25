# ros-realtime-skills

Agent skills (`SKILL.md`) for real-time ROS 2 development.

The skills encode hard real-time programming rules, `PREEMPT_RT` Linux constraints,
deterministic ROS 2 patterns, and eBPF and MC/DC verification, so that AI coding
agents generate deterministic, safety-aligned robotics code by default.

## Installation

The skills use the standard `SKILL.md` layout (`skills/<name>/SKILL.md` with `name`
and `description` frontmatter). Install them with the [`skills`](https://skills.sh)
CLI from a local checkout of this directory:

```bash
npx skills add /path/to/ros-realtime-skills                       # all skills
npx skills add /path/to/ros-realtime-skills --skill ros2-realtime # one skill
npx skills add /path/to/ros-realtime-skills --list                # what is available
```

Add `-a kiro-cli` (or another agent) to choose the target agent.

## Included Skills

| Skill | Scope | Core Enforced Invariants |
| :--- | :--- | :--- |
| **`hard-realtime-systems`** | OS-agnostic & Linux `PREEMPT_RT` user-space C++20 / Rust | `mlockall()`, `SCHED_FIFO` priority ceilings, priority-inheritance mutexes (`PTHREAD_PRIO_INHERIT`), zero heap allocations on hot paths, lock-free SPSC/MPMC ring buffers, CPU core partitioning, and IRQ affinity pinning. |
| **`ros2-realtime`** | Deterministic ROS 2 & `ros2_control` | `StaticSingleThreadedExecutor`, TLSF real-time allocators, bounded `KEEP_LAST` QoS, zero-copy `Iceoryx2` / `Zenoh` transport, `ros2_control` hardware interface separation, and safety-supervisor command clamping. |
| **`realtime-verification-mcdc-ebpf`** | Observability, Benchmarking & Safety CI | Native Linux eBPF (`aya` Rust / `USDT`) instead of out-of-tree LTTng modules, `cyclictest` / AI contention gates, and atomic boolean conditions for GCC/Clang/Rust MC/DC coverage. |

## Related Projects

Projects of the [ROS 2 Real-Time Working Group](https://github.com/ros-realtime):

* [`linux-real-time-kernel-builder`](https://github.com/ros-realtime/linux-real-time-kernel-builder): build and set up an RT kernel for ROS 2 testing.
* [`reference-system`](https://github.com/ros-realtime/reference-system): a reference system for comparing executors and other settings.
* [`ros2-realtime-examples`](https://github.com/ros-realtime/ros2-realtime-examples): minimal ROS 2 real-time cookbook recipes.
* [`ros2_realtime_benchmarks`](https://github.com/ros-realtime/ros2_realtime_benchmarks): ROS 2 real-time benchmark results.
* [`ros-realtime-rpi4-image`](https://github.com/ros-realtime/ros-realtime-rpi4-image): a Raspberry Pi image with ROS 2 and Linux RT preinstalled.

## CI Validation

`.github/workflows/validate-skills.yml` (run manually from the Actions tab) checks that each `skills/<name>/SKILL.md`
starts with YAML frontmatter containing `name: <name>` (matching its directory) and a
`description:`. It does not compile or lint the code blocks.

## License

Apache-2.0. See [`LICENSE`](LICENSE).
