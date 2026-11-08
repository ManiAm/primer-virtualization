# Quick Emulator (QEMU)

QEMU is an open-source machine emulator and virtualizer. It can emulate entire hardware platforms — including CPUs, memory, storage, and peripherals — allowing complete operating systems to run inside a virtual environment. QEMU is one of the most versatile tools in the virtualization ecosystem because it supports multiple operating modes, CPU architectures, and virtualization backends.

## Operating Modes

QEMU operates in three distinct modes:

### System Emulation

QEMU emulates an entire hardware platform, including the CPU, memory, storage, and peripheral devices. This enables running complete operating systems designed for different architectures — for example, booting an ARM-based OS on an x86 host.

System emulation provides a self-contained virtual environment that behaves like a physical machine, making it ideal for development, testing, and system-level experimentation. Later chapters cover system emulation for [x86](08_qemu_system_emulation.md) and [ARM](09_qemu_system_emulation_arm.md) in detail.

### User-mode Emulation

In this mode, QEMU runs individual user-space programs compiled for a different CPU architecture — without emulating an entire system. For example, an ARM binary can be executed on an x86 host without recompilation. This is particularly useful for debugging or testing cross-compiled applications. User-mode emulation is covered in [Chapter 12](12_qemu_user_emulation.md).

### Virtualization (with KVM and Other Backends)

QEMU can pair with hardware-assisted virtualization backends to run VMs at near-native speed. The most common pairing is with **KVM** (covered in the [previous chapter](03_type1_hypervisors.md#kernel-based-virtual-machine-kvm)), where KVM handles CPU and memory virtualization via hardware extensions (VT-x/AMD-V) while QEMU provides the device model — emulating network cards, storage controllers, display adapters, and other peripherals.

Besides KVM, QEMU also supports other backends:

- **Xen** — Open-source Type 1 hypervisor
- **HVF** — Apple's Hypervisor Framework on macOS
- **WHPX** — Windows Hypervisor Platform on Windows

## Dynamic Binary Translation

When QEMU operates in emulation mode (system or user-mode), it uses **dynamic binary translation** to convert guest CPU instructions into host CPU instructions on the fly. Rather than interpreting each guest instruction one at a time — which would be very slow — QEMU translates entire blocks of guest code into equivalent host code and caches the result. If the same guest code block executes again, the cached translation is reused without repeating the translation step.

When the host and guest use the **same architecture** (e.g., x86 on x86), the translation is relatively lightweight since most instructions map directly. When the architectures **differ** (e.g., ARM guest on x86 host), translation is more complex due to differences in instruction sets, register models, and memory ordering. This adds overhead from managing instruction set differences, handling system calls, and emulating hardware interfaces.

> **Note:** When QEMU is paired with KVM, dynamic binary translation is **not used** for guest CPU code. KVM executes guest instructions directly on the host CPU via hardware virtualization extensions, achieving near-native performance. QEMU's translation engine is only active when running without a hardware virtualization backend.

## Tiny Code Generator (TCG)

TCG is the component inside QEMU that performs dynamic binary translation. It replaced an older approach called *dyngen*, offering better performance, flexibility, and maintainability.

The translation pipeline works in three stages:

1. **Guest → Intermediate Representation (IR)** — QEMU translates a block of guest instructions into a platform-independent intermediate representation. The IR captures the semantics of the guest code without being tied to any specific host architecture.

2. **IR Optimization** — The IR is optimized before generating host code. Common optimizations include constant folding, dead code elimination, and basic block reordering.

3. **IR → Host Code** — TCG translates the optimized IR into native host machine code. The generated code is cached so that repeated execution of the same guest block skips the translation entirely.

During execution, QEMU intercepts and handles system calls, memory accesses, and hardware interactions as needed.

---

[← Previous: Type 1 Hypervisors](03_type1_hypervisors.md) | [Next: Virtual Disk Images →](05_virtual_disk_image.md)
