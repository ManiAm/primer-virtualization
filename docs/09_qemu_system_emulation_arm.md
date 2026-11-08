# QEMU System Emulation (ARM)

The [previous chapter](08_qemu_system_emulation.md) emulated an x86 VM on an x86 host — the guest and host shared the same architecture. This chapter demonstrates **cross-architecture emulation**: running an ARM64 (AArch64) virtual machine on an x86 host using QEMU's dynamic binary translation.

Both x86 and ARM64 VMs need firmware to boot. The difference is that QEMU **bundles SeaBIOS** — a legacy BIOS implementation — as the default firmware for x86, so x86 VMs boot out of the box without any extra setup. For ARM64, **no default firmware is included**. There is no legacy BIOS equivalent for the ARM architecture, so UEFI firmware must be installed separately and provided explicitly.

## Creating an ARM64 Virtual Machine

### Step 1 — Download the ISO and Create a Disk Image

Download the latest Alpine **AArch64** ISO from the [Alpine Linux downloads page](https://alpinelinux.org/downloads/).

Create a 2 GB QCOW2 disk image:

```bash
qemu-img create -f qcow2 alpine_arm.qcow2 2G
```

### Step 2 — Install UEFI Firmware

On x86 VMs, QEMU uses its built-in BIOS or UEFI firmware to boot. On ARM64, no firmware is bundled — it must be provided explicitly.

Debian/Ubuntu provide a pre-packaged UEFI firmware for AArch64:

```bash
sudo apt install qemu-efi-aarch64
```

This places a `QEMU_EFI.fd` file in `/usr/share/qemu-efi-aarch64/`.

### Step 3 — Boot from the ISO

```bash
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a57 \
    -m 2048 \
    -hda alpine_arm.qcow2 \
    -cdrom alpine-standard-aarch64.iso \
    -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd \
    -nographic
```

| Option        | Description                                                       |
|---------------|-------------------------------------------------------------------|
| `-M virt`     | Use the generic ARM virtual machine platform                      |
| `-cpu cortex-a57` | Emulate an ARM Cortex-A57 processor                          |
| `-m 2048`     | Allocate 2 GB of RAM                                              |
| `-hda`        | Attach the QCOW2 image as the hard disk                          |
| `-cdrom`      | Attach the ISO as a virtual CD-ROM                                |
| `-bios`       | Load the UEFI firmware from the specified file                    |
| `-nographic`  | Disable graphical output; use the terminal for console I/O        |

### Step 4 — Install and Boot from Disk

Follow the same installation process as the x86 walkthrough: run `setup-alpine`, select the QEMU hard disk, and choose `sys` to install permanently. Shut down with `poweroff` when finished.

On subsequent boots, the GRUB menu (installed by Alpine) allows booting directly from the hard disk.

## pflash: Persistent Firmware Storage

The `-bios` option used above loads firmware into memory at boot time but does not persist any changes the firmware makes (such as UEFI boot variables). For persistent firmware storage, QEMU supports the **pflash** (parallel NOR flash) device — an emulated flash memory chip commonly used to store UEFI firmware in virtual machines.

### Why pflash Is Needed

UEFI firmware expects two flash regions:

- **Code region** — The firmware binary itself (read-only)
- **Variable region** — UEFI variables such as boot order and Secure Boot settings (read-write, persisted across reboots)

Using `-bios` merges both into a single read-only load, which works for basic booting but does not support persistent UEFI variables. The pflash approach gives the firmware writable storage that survives VM restarts.

### Creating a pflash Image

Create a blank 64 MB raw image to serve as the firmware flash:

```bash
dd if=/dev/zero of=flash.img bs=1M count=64
```

Copy the UEFI firmware binary into the beginning of the flash image:

```bash
dd if=/usr/share/qemu-efi-aarch64/QEMU_EFI.fd of=flash.img conv=notrunc
```

The `conv=notrunc` flag ensures the output file retains its full 64 MB size — without it, `dd` would truncate the file to the size of the firmware binary.

### Booting with pflash

Replace `-bios` with a pflash drive:

```bash
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a57 \
    -m 2048 \
    -hda alpine_arm.qcow2 \
    -cdrom alpine-standard-aarch64.iso \
    -drive if=pflash,format=raw,file=flash.img \
    -nographic
```

The `-drive if=pflash` option attaches `flash.img` as a flash memory device. QEMU's UEFI firmware reads from and writes to this image, so boot variables and configuration changes persist across reboots.

---

[← Previous: System Emulation (x86)](08_qemu_system_emulation.md) | [Next: Direct Linux Boot →](10_direct_linux_boot.md)
