# QEMU System Emulation (x86)

QEMU system emulation creates a complete virtual machine — CPU, memory, storage, and peripherals — allowing a full operating system to boot inside it. This chapter covers installing QEMU, creating an x86 virtual machine, and common runtime options.

## Installing QEMU

On Ubuntu, install the system emulation package:

```bash
sudo apt install qemu-system
```

This installs emulator binaries for all supported architectures (`qemu-system-x86_64`, `qemu-system-aarch64`, `qemu-system-arm`, `qemu-system-riscv64`, and many more). For x86 emulation on an x86 host, the relevant binary is `qemu-system-x86_64`:

```bash
qemu-system-x86_64 --version
```

```
QEMU emulator version 8.0.4 (Debian 1:8.0.4+dfsg-1ubuntu3.23.10.5)
Copyright (c) 2003-2022 Fabrice Bellard and the QEMU Project developers
```

## Creating an x86 Virtual Machine

This walkthrough creates an **Alpine Linux** VM. Alpine is a lightweight, security-oriented Linux distribution — its small footprint makes it ideal for learning QEMU without long download or install times.

### Step 1 — Download the ISO

Download the latest Alpine x86_64 ISO from the [Alpine Linux downloads page](https://alpinelinux.org/downloads/).

### Step 2 — Create a Disk Image

Create a 2 GB QCOW2 disk image to serve as the VM's hard drive:

```bash
qemu-img create -f qcow2 alpine_x86.qcow2 2G
```

Since QCOW2 is a sparse format, the file starts small and grows as the guest writes data — so choosing a generous size has no immediate cost.

### Step 3 — Boot from the ISO

```bash
qemu-system-x86_64 -m 512 -hda alpine_x86.qcow2 -cdrom alpine-standard-x86_64.iso -boot d
```

| Option     | Description                                        |
|------------|----------------------------------------------------|
| `-m 512`   | Allocate 512 MB of RAM to the VM                   |
| `-hda`     | Attach the QCOW2 image as the first hard disk      |
| `-cdrom`   | Attach the ISO as a virtual CD-ROM drive           |
| `-boot d`  | Boot from the CD-ROM (see boot device table below) |

**Boot device letters:**

| Letter | Device                  |
|--------|-------------------------|
| `a`    | First floppy disk drive |
| `b`    | Second floppy disk drive|
| `c`    | First hard disk         |
| `d`    | CD-ROM drive            |
| `n`    | Network (PXE boot)      |

QEMU opens a graphical window and boots the Alpine installer. When the login prompt appears, log in as `root` (no password required).

### Step 4 — Install Alpine to Disk

At the Alpine shell, run the installer:

```bash
setup-alpine
```

When the script reaches the **Disk & Install** section, select the QEMU hard disk and choose `sys` to permanently install the OS. When the installation is complete, **do not reboot** — instead, shut down cleanly:

```bash
poweroff
```

### Step 5 — Boot from Disk

The OS is now installed on the QCOW2 image. Boot directly from the hard disk:

```bash
qemu-system-x86_64 -m 512 -hda alpine_x86.qcow2 -boot c
```

The `-boot c` option tells QEMU to boot from the first hard disk instead of the CD-ROM.

## Disabling Graphical Output

The `-nographic` option disables the QEMU graphical window and redirects all VM input and output to the terminal where QEMU was started:

```bash
qemu-system-x86_64 -m 512 -hda alpine_x86.qcow2 -boot c -nographic
```

This is useful when running QEMU over SSH or in environments without a display. To exit, press `Ctrl+A` followed by `X`.

## Redirecting the Serial Console

The serial port can be redirected to a telnet server, allowing remote access to the VM's console from a separate terminal:

```bash
qemu-system-x86_64 -m 512 -hda alpine_x86.qcow2 -boot c -nographic \
    -serial telnet:127.0.0.1:4567,server,nowait
```

This starts a telnet server on `127.0.0.1:4567`. Connect to it from another terminal:

```bash
telnet 127.0.0.1 4567
```

This is useful for debugging, monitoring, or scripting interactions with the VM's serial console without using the QEMU graphical window.

---

[← Previous: Minimal Root Filesystem](07_minimal_root_fs.md) | [Next: System Emulation (ARM) →](09_qemu_system_emulation_arm.md)
