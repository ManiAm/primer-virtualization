# Virtual Disk Image

A virtual disk image is a file that emulates a physical disk drive. It allows a virtual machine (VM) to store its operating system, applications, and data — just like a real hard disk. Depending on the format, disk images can support features such as snapshots, compression, encryption, and dynamic allocation.

Different virtualization platforms use different disk image formats:

| Format            | Year      | Platform(s)                     | Description                                                                            |
|-------------------|-----------|---------------------------------|----------------------------------------------------------------------------------------|
| **RAW**           | Pre-1990s | Generic (All)                   | Unformatted disk image — a direct byte-for-byte copy of a disk                         |
| **VMDK**          | 2001      | VMware                          | Virtual Machine Disk format used by VMware Workstation, ESXi, and vSphere              |
| **VHD**           | 2003      | Microsoft (Virtual PC, Hyper-V) | Virtual Hard Disk format for Microsoft's virtualization platforms                       |
| **QCOW**          | 2004      | QEMU                            | Initial version of QEMU Copy-On-Write format                                           |
| **QCOW2**         | 2006      | QEMU/KVM                        | Improved QEMU format — supports snapshots, compression, and encryption                 |
| **VDI**           | 2007      | VirtualBox                      | Virtual Disk Image format used by Oracle VirtualBox                                    |
| **VHDX**          | 2012      | Microsoft Hyper-V               | Enhanced version of VHD with better performance and larger disk support (up to 64 TB)  |
| **XVA**           | 2013      | XenServer                       | XenServer format for VM export/import                                                  |
| **Parallels HDD** | 2013      | Parallels Desktop               | Virtual disk format used by Parallels for macOS virtualization                         |

This chapter focuses on **raw images** — the simplest format. The [next chapter](06_qcow2_image.md) covers QCOW2 in detail.

## Raw Image

A raw image is a simple, uncompressed disk image format that stores a direct byte-for-byte representation of a disk. It contains no metadata, compression, or management features.

**Advantages:**
- Fastest read/write performance — no processing overhead
- Compatible with virtually all virtualization platforms and tools

**Limitations:**
- No snapshots, dynamic resizing, or compression
- Consumes as much disk space as its defined size (unless created as a sparse file)

Raw images typically have a `.img` or `.raw` extension.

### Sparse vs. Fully Allocated

A key concept when working with raw images is **sparse files**. A sparse file reports a large logical size but only consumes physical disk space for the blocks that actually contain data. Blocks that have never been written are not stored — the filesystem generates zero bytes on the fly when they are read.

For example, a 1 GB sparse file might occupy only a few kilobytes on disk until data is written to it. This saves storage but does not guarantee that space will be available later if the disk fills up.

## Creating Raw Images

There are several tools for creating raw disk images, each with different trade-offs:

| Command       | Sparse | Space Preallocated | Speed     | Typical Use Case                               |
|---------------|--------|--------------------|-----------|-------------------------------------------------|
| `dd`          | No     | Yes                | Slow      | Creating bootable images, full disk writes      |
| `truncate`    | Yes    | No                 | Very fast | Quick sparse file creation                      |
| `fallocate`   | Yes    | Yes                | Very fast | Preallocating space efficiently for disk images |
| `qemu-img`    | Yes    | No (by default)    | Fast      | Creating VM disk images in various formats      |

The following sections demonstrate each tool by creating a 1 GB raw image called `mydisk.img`.

### `dd`

```bash
dd if=/dev/zero of=mydisk.img bs=1M count=1024
```

- `if=/dev/zero` — Read from `/dev/zero`, a special file that provides an infinite stream of zero bytes
- `bs=1M` — Set the block size to 1 megabyte
- `count=1024` — Write 1,024 blocks (1,024 × 1 MB = 1 GB)

This creates a **fully allocated** image. Every byte is physically written to disk, which guarantees space availability but is slow.

Verify with `stat`:

```bash
stat mydisk.img
```

```
Size: 1073741824  Blocks: 2097160  IO Block: 4096  regular file
```

The file has 2,097,160 blocks allocated — all storing zero bytes copied from `/dev/zero`, all occupying real disk space.

### `truncate`

```bash
truncate -s 1G mydisk.img
```

This creates a **sparse** file. The file appears as 1 GB but consumes no actual disk space until data is written.

```bash
stat mydisk.img
```

```
Size: 1073741824  Blocks: 0  IO Block: 4096  regular file
```

The logical size is 1 GB, but block count is **0** — no physical storage is consumed. When the file is read, the filesystem dynamically returns zero bytes for any unwritten regions.

`truncate` is the fastest option, but since no space is reserved, writes may fail later if the disk fills up. Sparse files can also lead to fragmentation as data is written over time.

### `fallocate`

```bash
fallocate -l 1G mydisk.img
```

This creates a **sparse file with preallocated space**. The filesystem reserves 1 GB of disk space without writing any data, which is nearly as fast as `truncate` but guarantees that the space is available for future writes.

`fallocate` is the best choice when you want both speed and space reliability.

### `qemu-img`

```bash
qemu-img create -f raw mydisk.img 1G
```

This creates a **sparse** raw image by default — similar to `truncate`.

```bash
stat mydisk.img
```

```
Size: 1073741824  Blocks: 8  IO Block: 4096  regular file
```

The minimal block count (8 blocks = 4 KB) is just filesystem metadata. To preallocate space, add `-o preallocation=full`.

Verify the image format and details:

```bash
qemu-img info mydisk.img
```

```
image: mydisk.img
file format: raw
virtual size: 1 GiB (1073741824 bytes)
disk size: 4 KiB
```

`qemu-img` is the standard tool for working with VM disk images and is used throughout the remaining chapters.

## Converting Between Raw and QCOW2

Raw images consume their full allocated size on disk and lack advanced features. QCOW2 images support compression, snapshots, and dynamic allocation, making them more storage-efficient for most use cases.

**Raw → QCOW2:**

```bash
qemu-img convert -f raw -O qcow2 mydisk.img mydisk.qcow2
```

To enable compression and further reduce file size:

```bash
qemu-img convert -f raw -O qcow2 -c mydisk.img mydisk.qcow2
```

**QCOW2 → Raw:**

```bash
qemu-img convert -f qcow2 -O raw mydisk.qcow2 mydisk.raw
```

Converting back to raw is useful when a hypervisor or tool requires the raw format.

After any conversion, verify the result:

```bash
qemu-img info mydisk.qcow2
```

---

[← Previous: QEMU](04_qemu.md) | [Next: QCOW2 Images →](06_qcow2_image.md)
