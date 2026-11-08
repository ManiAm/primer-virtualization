# QEMU User-mode Emulation

[Chapter 4](04_qemu.md) introduced QEMU's three operating modes. **System emulation** (covered in chapters [8](08_qemu_system_emulation.md) and [9](09_qemu_system_emulation_arm.md)) emulates an entire hardware platform to run a complete OS. **User-mode emulation** takes a lighter approach — it runs a single Linux binary compiled for a different CPU architecture, without emulating any hardware. QEMU translates the binary's CPU instructions on the fly and forwards its system calls to the host kernel.

This is useful for testing cross-compiled applications, running foreign-architecture tools, and debugging — all without booting a full virtual machine.

## Cross-Compiling a Test Program

To demonstrate user-mode emulation, start with a simple C program:

```c
#include <stdio.h>

int main() {
    printf("Hello, ARM!\n");
    return 0;
}
```

Save it as `hello.c`. To compile this for ARM64 instead of the host's x86, install the ARM64 cross-compiler:

```bash
sudo apt install gcc-aarch64-linux-gnu
```

A **cross-compiler** produces executables for a different architecture than the one it runs on. Here, `aarch64-linux-gnu-gcc` runs on x86 but generates ARM64 binaries.

Compile the program with **static linking** (all libraries are embedded in the binary):

```bash
aarch64-linux-gnu-gcc -static -o hello-arm hello.c
```

## Running the Binary

Attempting to run the ARM64 binary directly on an x86 host fails:

```bash
./hello-arm
```

```
bash: ./hello-arm: cannot execute binary file: Exec format error
```

This is expected — the binary contains ARM64 instructions that the x86 CPU cannot execute. User-mode emulation solves this.

## Installing QEMU User-mode Emulation

Install the QEMU user-mode package:

```bash
sudo apt install qemu-user
```

This installs emulators for all supported architectures (`qemu-aarch64`, `qemu-arm`, `qemu-riscv64`, etc.). You can now run the ARM64 binary by invoking the appropriate emulator explicitly:

```bash
qemu-aarch64 ./hello-arm
```

```
Hello, ARM!
```

## Transparent Execution with binfmt_misc

After installing `qemu-user`, the ARM64 binary can also be run directly — without specifying `qemu-aarch64`:

```bash
./hello-arm
```

```
Hello, ARM!
```

This works because the package registers QEMU with **binfmt_misc**, a Linux kernel feature that associates binary formats with user-space interpreter programs. The name stands for "binary format miscellaneous." When the kernel encounters an executable it cannot run natively, it inspects the binary's header (the ELF magic bytes and architecture field), finds the registered handler for that format, and launches it automatically. In this case, the kernel detects an ARM64 ELF binary and invokes `qemu-aarch64` behind the scenes.

## Dynamic Linking

The example above used **static linking** (`-static`), which embeds all library code into the binary itself. This makes the binary self-contained but larger. Without `-static`, the compiler produces a **dynamically linked** binary that depends on shared libraries at runtime:

```bash
aarch64-linux-gnu-gcc -o hello-arm hello.c
```

Running this binary fails with a different error:

```bash
qemu-aarch64 ./hello-arm
```

```
/lib/ld-linux-aarch64.so.1: No such file or directory
```

The binary expects the ARM64 dynamic linker (`ld-linux-aarch64.so.1`) and shared libraries (like `libc.so`), but these do not exist on an x86 host. QEMU needs to be told where to find them.

The cross-compiler package installs ARM64 libraries under `/usr/aarch64-linux-gnu/`. Point QEMU to this directory with the `-L` flag, which sets the root filesystem path for library resolution:

```bash
qemu-aarch64 -L /usr/aarch64-linux-gnu ./hello-arm
```

```
Hello, ARM!
```

QEMU prepends the `-L` path to all library lookups, so `/lib/ld-linux-aarch64.so.1` resolves to `/usr/aarch64-linux-gnu/lib/ld-linux-aarch64.so.1`.

---

[← Previous: QEMU and KVM](11_qemu_and_kvm.md) | [Next: QEMU Monitor →](13_qemu_monitor.md)
