# Host-Guest File Sharing

Transferring files between the host and a QEMU virtual machine is a common need — whether for injecting configuration files, extracting build artifacts, or sharing a working directory. QEMU supports several approaches, each with different trade-offs in performance, flexibility, and setup complexity.

## Comparison of Sharing Methods

The following table compares all available methods across five key criteria:

| Category         | Method           | Bi-directional | R/W Access | Performance | Networking Required | Guest Support Required |
|------------------|------------------|----------------|------------|-------------|---------------------|------------------------|
| **Manual**       | USB Passthrough  | Yes            | Yes        | Medium      | No                  | No                     |
| **Manual**       | ISO/CD-ROM       | Host → Guest   | Read-only  | High        | No                  | No                     |
| **Manual**       | SCP / Rsync      | Yes            | Yes        | Medium      | Yes                 | SSH server/client      |
| **Block Device** | Raw Disk         | Yes            | Yes        | High        | No                  | No                     |
| **Block Device** | QCOW2 Image      | Yes            | Yes        | High        | No                  | No                     |
| **Network**      | NFS              | Yes            | Yes        | High        | Yes                 | NFS client             |
| **Network**      | SMB (Samba)      | Yes            | Yes        | High        | Yes                 | SMB/CIFS client        |
| **Network**      | SSHFS            | Yes            | Yes        | Medium      | Yes                 | SSH + FUSE             |
| **Network**      | QEMU TFTP        | Host → Guest   | Read-only  | Low         | Yes (user-mode)     | TFTP client            |
| **Paravirt**     | 9pfs             | Yes            | Yes        | Medium      | No                  | 9p kernel module       |
| **Paravirt**     | Virtio-FS        | Yes            | Yes        | High        | No                  | virtiofs driver (5.4+) |

- **Bi-directional** — Whether both the host and guest can exchange data through the same mechanism. Methods marked "Host → Guest" only expose host files to the guest; the host cannot access guest-side files through the same interface.

- **R/W Access** — Whether the guest can both read and write to the shared resource. Read-only methods (ISO, TFTP) are suitable for static content delivery.

- **Performance** — Relative throughput and latency. Methods using direct hardware channels (Virtio-FS, raw disk) offer the best performance. Network-based methods add protocol overhead.

- **Networking Required** — Whether the method depends on network connectivity between host and guest. Paravirtualized and block-based methods work through virtual hardware channels without any network stack.

- **Guest Support Required** — Whether the guest OS needs specific drivers, kernel modules, or user-space tools beyond a stock installation.

**Quick guide:**

- **Best performance** — Virtio-FS (Linux guests)
- **Simple built-in support** — 9pfs
- **Network-based sharing** — NFS or SMB
- **Persistent storage** — Block devices or QCOW2 images
- **One-time transfers** — USB, SCP, or ISO images

The following sections demonstrate each method with practical examples.

## USB Passthrough

USB passthrough assigns a physical USB device from the host directly to the guest. The guest OS interacts with it as if it were plugged in natively — useful for storage drives, input devices, or hardware dongles.

### Identifying the Device

List USB devices on the host:

```bash
lsusb
```

```
Bus 002 Device 002: ID 0781:5581 SanDisk Corp. Ultra
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
```

The `ID` field shows `VendorID:ProductID`. For the SanDisk drive above, that is `0781:5581`.

### Passing the Device to QEMU

```bash
sudo qemu-system-x86_64 \
    -m 512 \
    -hda alpine_x86.qcow2 \
    -boot c \
    -usb \
    -device usb-host,vendorid=0x0781,productid=0x5581
```

| Option          | Description                                                  |
|-----------------|--------------------------------------------------------------|
| `-usb`          | Enable the USB controller in the guest                       |
| `-device usb-host,...` | Pass the specified USB device through to the guest    |

> **Note:** `sudo` is required because accessing host USB devices needs root privileges.

Once the VM boots, verify the device is visible inside the guest:

```bash
lsusb
```

```
Bus 001 Device 001: ID 1d6b:0001
Bus 001 Device 002: ID 0781:5581
```

If the device is a storage drive, mount it:

```bash
mkdir /mnt/usb
mount /dev/sdb /mnt/usb
```

## ISO/CD-ROM Sharing

An ISO image can be attached to a VM as a read-only virtual CD-ROM — a simple way to transfer a batch of files without networking.

### Creating an ISO Image

Install `genisoimage` and create an ISO from files on the host:

```bash
sudo apt install genisoimage
genisoimage -o my_image.iso file1.txt file2.txt
```

For better cross-platform compatibility, add extension flags:

| Option | Name                  | Purpose                                                 |
|--------|-----------------------|---------------------------------------------------------|
| `-r`   | Rock Ridge (modified) | Unix compatibility with permissions reset to root       |
| `-R`   | Rock Ridge (strict)   | Unix compatibility preserving original file ownership   |
| `-J`   | Joliet                | Windows-compatible long filenames                       |

### Attaching the ISO

Pass the ISO as a second CD-ROM drive alongside the boot media:

```bash
qemu-system-x86_64 -m 512 \
    -hda alpine_x86.qcow2 \
    -cdrom alpine-standard-3.20.1-x86_64.iso \
    -boot d \
    -drive file=my_image.iso,media=cdrom
```

| Option                              | Description                                    |
|-------------------------------------|------------------------------------------------|
| `-cdrom`                            | Attach the primary bootable ISO                |
| `-boot d`                           | Boot from the primary CD-ROM                   |
| `-drive file=...,media=cdrom`       | Attach an additional ISO as a second CD-ROM    |

### Mounting Inside the Guest

List available CD-ROM devices and mount the additional ISO:

```bash
ls /dev/sr*
```

```
/dev/sr0  /dev/sr1
```

```bash
mkdir -p /mnt/extra_iso
mount /dev/sr0 /mnt/extra_iso
ls /mnt/extra_iso/
```

```
file1.txt   file2.txt
```

## Raw Disk Passthrough

Raw disk passthrough gives the guest direct access to a physical disk or partition on the host. The guest sees it as a real drive and can partition, format, and mount it.

Identify the target disk on the host:

```bash
lsblk
```

```
sda      8:0    0    25G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0    25G  0 part /
sdb      8:16   1  28.7G  0 disk
```

Pass `/dev/sdb` to the guest:

```bash
sudo qemu-system-x86_64 -m 512 \
    -hda alpine_x86.qcow2 \
    -boot c \
    -device virtio-blk-pci,drive=disk0 \
    -drive file=/dev/sdb,if=none,id=disk0,format=raw
```

| Option                          | Description                                             |
|---------------------------------|---------------------------------------------------------|
| `-device virtio-blk-pci,...`    | Create a virtio block device in the guest               |
| `-drive file=/dev/sdb,...`      | Attach the host's physical disk as the backing storage  |

> **Note:** `sudo` is required for raw device access. Be careful — the guest has full read-write access to the disk.

Inside the guest, list partitions and mount:

```bash
fdisk -l
mount /dev/vda /mnt
```

## Virtual Disk Image

A QCOW2 image can serve as a shared storage medium: populate it with files on the host, then attach it to the VM. This approach uses the same NBD (Network Block Device) workflow covered in [Chapter 7 — Minimal Root Filesystem](07_minimal_root_fs.md).

### Preparing the Image on the Host

Create a small disk image, format it, and copy files into it:

```bash
qemu-img create -f qcow2 shared.img 10M

sudo modprobe nbd
sudo qemu-nbd --connect=/dev/nbd0 shared.img
sudo mkfs.ext4 /dev/nbd0

sudo mkdir -p /mnt/qemu-share
sudo mount /dev/nbd0 /mnt/qemu-share
sudo cp file1.txt file2.txt /mnt/qemu-share/

sudo umount /mnt/qemu-share
sudo qemu-nbd --disconnect /dev/nbd0
```

### Attaching the Image to the VM

```bash
qemu-system-x86_64 -m 512 \
    -hda alpine_x86.qcow2 \
    -boot c \
    -drive file=shared.img,if=virtio
```

The `-drive file=shared.img,if=virtio` option attaches the image using the virtio interface, which provides high-performance paravirtualized storage.

Inside the guest, the image appears as `/dev/vda`:

```bash
mount /dev/vda /mnt
ls /mnt/
```

```
file1.txt   file2.txt
```

> **Important:** Only the host or the guest can mount the image at a time — not both simultaneously. If the host attempts to connect via NBD while QEMU is running, it will fail with a lock error:
> ```
> qemu-nbd: Failed to blk_new_open 'shared.img': Failed to get "write" lock
> Is another process using the image [shared.img]?
> ```

## QEMU's Built-in TFTP Server

QEMU's user-mode networking stack includes a simple TFTP server. It requires no setup on the host — just point it to a directory and the guest can download files from it.

```bash
qemu-system-x86_64 -m 512 \
    -hda alpine_x86.qcow2 \
    -boot c \
    -netdev user,id=net0,tftp=/path/to/shared_dir \
    -device e1000,netdev=net0
```

| Option                        | Description                                                |
|-------------------------------|------------------------------------------------------------|
| `tftp=/path/to/shared_dir`    | Host directory to serve via TFTP                           |
| `-device e1000,netdev=net0`   | Attach an emulated Intel e1000 NIC to the guest            |

Inside the guest, retrieve files using a TFTP client. QEMU's default gateway address is `10.0.2.2`:

```bash
tftp 10.0.2.2 -c get filename.txt
```

> **Note:** The TFTP server is read-only and unencrypted. It is best suited for simple bootstrapping or file transfers in controlled environments.

## Plan 9 File System (9pfs)

9pfs is a lightweight, network-transparent file sharing protocol originally developed for the Plan 9 operating system. QEMU supports it natively via the `-virtfs` option, making it one of the easiest methods for continuous host-guest file sharing. Unlike Virtio-FS, 9pfs does not require a separate daemon — it is built directly into QEMU.

### Setup

Create a shared directory on the host and start the VM:

```bash
mkdir -p ~/shared_dir

qemu-system-x86_64 -m 512 \
    -hda alpine_x86.qcow2 \
    -boot c \
    -virtfs local,id=myfs,path=$HOME/shared_dir,mount_tag=sharedfolder,security_model=mapped
```

| Option                     | Description                                              |
|----------------------------|----------------------------------------------------------|
| `-virtfs local,...`        | Enable 9pfs with a local directory backend               |
| `path=...`                 | Host directory to share                                  |
| `mount_tag=sharedfolder`   | Tag the guest uses to identify this share when mounting   |
| `security_model=mapped`    | How file ownership and permissions are handled            |

### Mounting Inside the Guest

```bash
mount -t 9p -o trans=virtio sharedfolder /mnt
```

Changes on either side are visible immediately. Create a file on the host:

```bash
echo "Hello from host" > ~/shared_dir/hello.txt
```

Read it from the guest:

```bash
cat /mnt/hello.txt
```

### Security Models

File permission handling depends on the `security_model` option:

| Security Model   | Behavior                                                                              |
|------------------|---------------------------------------------------------------------------------------|
| **mapped**       | Guest file metadata (UID, GID, permissions) is stored in host extended attributes. Guest-created files appear owned by the QEMU process on the host. |
| **passthrough**  | Guest UID/GID is preserved exactly on the host. Requires QEMU to run as root.         |
| **none**         | All files appear owned by the QEMU process user. Simplest setup but no permission isolation. |

## Virtio-FS

Virtio-FS is a high-performance file sharing mechanism designed specifically for QEMU/KVM environments. It leverages FUSE (Filesystem in Userspace) and the virtio transport to provide fast, low-latency access to host directories — bypassing the guest's network stack entirely. Virtio-FS also supports DAX (Direct Access), which allows the guest to memory-map host files with zero-copy overhead.

Compared to 9pfs, Virtio-FS offers significantly better performance and caching, but requires a separate daemon (`virtiofsd`) running on the host.

### Starting the virtiofsd Daemon

Install the required packages:

```bash
sudo apt install virtiofsd uidmap -y
```

Create a shared directory and start the daemon:

```bash
sudo mkdir -p /mnt/shared
sudo chown $(whoami) /mnt/shared

sudo /usr/lib/qemu/virtiofsd \
    --socket-path=/tmp/vhostqemu \
    --shared-dir=/mnt/shared
```

The daemon listens on a Unix socket and waits for QEMU to connect.

### Starting the VM

In a second terminal, launch QEMU with Virtio-FS support:

```bash
sudo qemu-system-x86_64 -m 512 \
    -hda alpine_x86.qcow2 \
    -boot c \
    -chardev socket,id=char0,path=/tmp/vhostqemu \
    -device vhost-user-fs-pci,chardev=char0,tag=myfs
```

| Option                                 | Description                                          |
|----------------------------------------|------------------------------------------------------|
| `-chardev socket,...`                  | Connect to the virtiofsd daemon's Unix socket        |
| `-device vhost-user-fs-pci,...`        | Create a Virtio-FS device in the guest               |
| `tag=myfs`                             | Mount tag the guest uses to identify this share       |

### Mounting Inside the Guest

```bash
mkdir /mnt/guest_shared
mount -t virtiofs myfs /mnt/guest_shared
```

The shared directory is now accessible in the guest with high-performance host-guest file access.

---

[← Previous: QEMU Machine Protocol (QMP)](14_qemu_qmp.md) | [Next: Virtual Network Adapters →](16_virtual_network_adapters.md)
