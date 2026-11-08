# Direct Linux Boot

Normally, a virtual machine follows the same boot sequence as physical hardware: firmware (BIOS or UEFI) initializes the hardware, loads a bootloader such as GRUB, and the bootloader then loads the Linux kernel. QEMU can bypass this entire chain by loading the kernel binary directly into memory, along with an optional initial RAM disk (initrd) and kernel command-line parameters.

The three QEMU options that enable direct boot are:

| Option     | Description                                                     |
|------------|-----------------------------------------------------------------|
| `-kernel`  | Path to the kernel image (e.g., `bzImage`)                      |
| `-initrd`  | Path to the initial RAM disk image                              |
| `-append`  | Kernel command-line parameters (e.g., `console=ttyS0 root=...`) |

This approach is useful for kernel development, testing, and debugging — it eliminates bootloader configuration and reduces the edit-compile-boot cycle to seconds.

## Obtaining a Kernel and initrd

Before using direct boot, you need two files: a **kernel image** and (optionally) an **initrd**. On most Linux systems, both are readily available in `/boot/`:

```bash
ls /boot/vmlinuz-*
ls /boot/initrd.img-*      # Debian/Ubuntu
ls /boot/initramfs-*       # Fedora/RHEL
```

Copy the matching pair to your working directory:

```bash
cp /boot/vmlinuz-$(uname -r) bzImage
cp /boot/initrd.img-$(uname -r) initrd.img    # adjust filename for your distro
```

> **Important:** The kernel and initrd must come from the same version. A mismatched initrd may fail to load the kernel modules it expects.

If you have compiled a custom kernel, use the resulting `bzImage` (typically at `arch/x86/boot/bzImage` in the kernel source tree). In that case, you will also need to generate a matching initrd — see [Generating a Custom initrd with Dracut](#generating-a-custom-initrd-with-dracut) below.

## Booting a Kernel Without initrd

To see direct boot in action, start with just a kernel and no root filesystem:

```bash
qemu-system-x86_64 -kernel bzImage -append "console=ttyS0" -nographic
```

The `console=ttyS0` parameter tells the kernel to use the first serial port as its console. This is necessary when running with `-nographic` so that kernel output appears in the terminal instead of a graphical window.

Without an initrd or root filesystem, the kernel will panic after completing its initialization:

```
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(8,0)
```

This is expected — the kernel booted successfully but could not find a root filesystem to mount. The next sections show two ways to solve this: booting into a temporary RAM disk, or mounting a persistent disk image.

## Generating a Custom initrd with Dracut

If you are using a pre-built kernel and initrd from `/boot/`, you can skip this section — those files are already paired and ready to use. However, if you compiled a custom kernel, or need a minimal initrd for testing (e.g., one containing only a bash shell), you will need to generate your own.

An **initrd** (initial RAM disk) is a small temporary filesystem loaded into memory at boot time. It provides the minimal environment — device drivers, filesystem tools, and an init script — needed to locate the real root filesystem and hand off control to the system's init process.

**Dracut** is a tool for generating initrd images, common on Red Hat-based distributions and available on Debian/Ubuntu:

```bash
sudo apt install dracut
```

Generate a minimal initrd that includes a bash shell:

```bash
sudo dracut --force --no-kernel --hostonly --no-hostonly-cmdline --add "bash" initrd.img
```

| Option                    | Description                                                                |
|---------------------------|----------------------------------------------------------------------------|
| `--force`                 | Overwrite an existing `initrd.img` if present                              |
| `--no-kernel`             | Exclude kernel modules — suitable when all required drivers are built into the kernel   |
| `--hostonly`              | Include only drivers needed for the current host hardware (produces a smaller image)  |
| `--no-hostonly-cmdline`   | Do not embed `/proc/cmdline` into the initrd — kernel parameters will be supplied via `-append` instead |
| `--add "bash"`            | Include the `bash` dracut module so an interactive shell is available inside the initrd |

To list all available dracut modules:

```bash
dracut --list-modules
```

To inspect the contents of the generated initrd:

```bash
lsinitrd initrd.img
```

## Booting with initrd

Whether you copied the initrd from `/boot/` or generated one with Dracut, you can pass it to QEMU alongside the kernel. This example boots into the initrd itself as a self-contained RAM environment — no disk image is needed:

```bash
qemu-system-x86_64 -kernel bzImage -initrd initrd.img \
    -append "console=ttyS0 rdinit=/bin/bash" -nographic
```

The `rdinit=/bin/bash` parameter tells the kernel to run `/bin/bash` as the init process inside the initrd, instead of the default `/init` script. This drops you into an interactive shell within the RAM disk environment — useful for testing or inspecting the initrd contents.

Since this is a RAM disk with no real init system (no systemd, no OpenRC), standard shutdown commands are unavailable. To restart, use:

```bash
/sbin/reboot -f
```

> **Note:** The initrd is a temporary in-memory filesystem. Any files created or changes made are lost when the VM reboots.

## Booting with a Persistent Disk Image

The previous section booted into a temporary RAM environment. For persistent storage, attach a QCOW2 disk image containing a root filesystem. The kernel boots using the initrd for early initialization (loading drivers and mounting the disk), then pivots to the disk image as the permanent root filesystem.

### Step 1 — Prepare the root filesystem

Create a minimal root filesystem in `rootfs.qcow2` as described in [Chapter 7 — Minimal Root Filesystem](07_minimal_root_fs.md).

### Step 2 — Obtain an initrd

Use the host's initrd from `/boot/` if available:

```bash
cp /boot/initrd.img-$(uname -r) initrd.img
```

Alternatively, generate one with Dracut. The `--add "bash"` flag is not needed here — the root filesystem on disk provides the shell and init system:

```bash
sudo dracut --force --no-kernel --hostonly --no-hostonly-cmdline initrd.img
```

### Step 3 — Boot with the disk image

```bash
qemu-system-x86_64 -kernel bzImage -initrd initrd.img -hda rootfs.qcow2 \
    -append "console=ttyS0 root=/dev/sda rw" -nographic
```

| Parameter       | Description                                                    |
|-----------------|----------------------------------------------------------------|
| `-hda`          | Attach `rootfs.qcow2` as the primary hard disk (`/dev/sda`)   |
| `root=/dev/sda` | Tell the kernel the root filesystem is on the first hard disk  |
| `rw`            | Mount the root filesystem as read-write                        |

Notice that `rdinit=/bin/bash` is **not** used here. With `root=/dev/sda`, the initrd's default init scripts take over — they load the necessary drivers, mount `/dev/sda` as the root filesystem, and hand off control to the init process on disk.

### Step 4 — Verify

Once booted, confirm the root filesystem is mounted from the disk:

```bash
bash
findmnt /
```

```
TARGET SOURCE   FSTYPE OPTIONS
/      /dev/sda ext4   rw,relatime
```

Verify the kernel command-line parameters:

```bash
cat /proc/cmdline
```

```
console=ttyS0 root=/dev/sda rw
```

Changes to the root filesystem now persist across reboots. To verify:

```bash
touch /test.txt
echo b > /proc/sysrq-trigger
```

The `echo b > /proc/sysrq-trigger` command sends a reboot request through the kernel's **SysRq** (System Request) mechanism — a set of low-level kernel commands accessible via `/proc/sysrq-trigger`. This is a quick way to reboot when no standard init system is available.

After the VM restarts, `/test.txt` will still be present.

## Direct Alpine Linux Boot (ARM64)

Direct boot also works for cross-architecture emulation. This example boots an Alpine Linux ARM64 system using pre-built kernel and initrd files — no firmware setup is required because the kernel is loaded directly by QEMU.

Download `vmlinuz-lts` and `initramfs-lts` from the [Alpine ARM64 netboot directory](https://dl-cdn.alpinelinux.org/alpine/edge/releases/aarch64/netboot/). For the disk image, use the `alpine_arm.qcow2` image created in [Chapter 9 — System Emulation (ARM)](09_qemu_system_emulation_arm.md):

```bash
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a57 \
    -m 512M \
    -hda alpine_arm.qcow2 \
    -kernel vmlinuz-lts \
    -initrd initramfs-lts \
    -append "console=ttyAMA0 ip=dhcp alpine_repo=http://dl-cdn.alpinelinux.org/alpine/edge/main/" \
    -nographic
```

Note that ARM uses `ttyAMA0` (the PL011 UART) as the serial console instead of x86's `ttyS0`.

---

[← Previous: System Emulation (ARM)](09_qemu_system_emulation_arm.md) | [Next: QEMU and KVM →](11_qemu_and_kvm.md)
