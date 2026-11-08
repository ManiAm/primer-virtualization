# Mounting a QCOW2 Image

Before a QCOW2 disk image can be used by a virtual machine, it needs a filesystem. This chapter walks through mounting a QCOW2 image on the host, formatting it, and populating it with a minimal Linux root filesystem — all without booting a VM.

## Creating and Connecting the Image

Create a 1 GB QCOW2 disk image:

```bash
qemu-img create -f qcow2 rootfs.qcow2 1G
```

To access the image as a block device on the host, use the **Network Block Device** (NBD) kernel module and `qemu-nbd`:

```bash
sudo modprobe nbd
sudo qemu-nbd --connect=/dev/nbd0 rootfs.qcow2
```

The `nbd` module exposes the QCOW2 image as `/dev/nbd0`, making it behave like a regular disk from the host's perspective.

## Formatting the Image

At this point the image is empty — it has no partition table or filesystem. Attempting to mount it directly will fail:

```bash
sudo mount /dev/nbd0 /mnt    # fails: no filesystem
```

Format the image with an ext4 filesystem:

```bash
sudo mkfs.ext4 /dev/nbd0
```

```
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done
Creating filesystem with 524288 4k blocks and 131072 inodes
Filesystem UUID: f3ed1369-70df-47a9-a928-691ffbc6e309
Superblock backups stored on blocks:
    32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

Verify the filesystem:

```bash
sudo blkid /dev/nbd0
```

## Mounting and Unmounting

The image can now be mounted like any block device:

```bash
sudo mount /dev/nbd0 /mnt
```

When finished, always unmount and disconnect in this order:

```bash
sudo umount /mnt
sudo qemu-nbd --disconnect /dev/nbd0
```

## Creating a Minimal Root Filesystem

With the image mounted, it can be populated with a minimal Linux root filesystem using `debootstrap`. This tool installs a base Debian/Ubuntu system into an arbitrary directory — no installer or VM required.

**Step 1** — Create, format, and mount the image as described above.

**Step 2** — Install a minimal Ubuntu base system (Focal Fossa) for the amd64 architecture:

```bash
sudo debootstrap --arch=amd64 --variant=minbase focal /mnt http://archive.ubuntu.com/ubuntu/
```

This downloads and installs the minimal set of packages needed for a functional Ubuntu system into `/mnt`.

**Step 3** — Verify the result. The image now contains a standard Linux directory structure:

```bash
ls -l /mnt
```

You should see the familiar top-level directories: `bin`, `etc`, `lib`, `usr`, `var`, and so on.

**Step 4** — Unmount and disconnect:

```bash
sudo umount /mnt
sudo qemu-nbd --disconnect /dev/nbd0
```

The `rootfs.qcow2` image now contains a bootable root filesystem and can be used with QEMU's system emulation, which is covered in the [next chapter](08_qemu_system_emulation.md).

---

[← Previous: QCOW2 Images](06_qcow2_image.md) | [Next: System Emulation (x86) →](08_qemu_system_emulation.md)
