# Virtual Network Adapters

A virtual network adapter is a software emulation of a physical Network Interface Card (NIC). It provides a VM with network connectivity — allowing communication with other VMs, the host system, and external networks. From the guest OS's perspective, a virtual NIC behaves identically to a real hardware NIC, complete with a MAC address, driver interface, and link state.

Virtualization platforms offer a choice of adapter types, each emulating a different real-world NIC model. The choice affects guest driver compatibility, performance, and available features.

## Emulated vs. Paravirtualized Adapters

As introduced in [Chapter 2](02_hardware_virtualization.md#io-virtualization-extensions), virtual I/O devices are either **emulated** (replicating real hardware for broad compatibility) or **paravirtualized** (using virtio for higher performance at the cost of requiring guest drivers).

The practical trade-off when choosing an adapter: emulated NICs work out of the box with any guest OS, while `virtio-net` requires virtio drivers — Linux includes them by default, but Windows guests need the [virtio-win driver package](https://github.com/virtio-win/virtio-win-pkg-scripts).

## QEMU Adapter Types

QEMU offers several virtual NIC models, selectable via the `-device` option (or the legacy `-net nic,model=` syntax). Common choices include:

| Adapter         | Description                                             | Best For                                          |
|-----------------|---------------------------------------------------------|---------------------------------------------------|
| `e1000`         | Emulates Intel 82540EM — widely supported 1 Gbps NIC    | Broad compatibility with modern OSes              |
| `e1000e`        | Emulates Intel 82574L — newer, more features            | Modern Windows and Linux guests                   |
| `rtl8139`       | Emulates Realtek 8139 — 100 Mbps Fast Ethernet          | Legacy OSes with built-in Realtek drivers         |
| `virtio-net-pci`| Paravirtualized virtio NIC                              | Maximum performance (Linux, modern Windows)       |

`virtio-net-pci` is the recommended choice when the guest supports virtio drivers. For legacy guests or maximum compatibility, `e1000` is a safe default.

To list all available NIC models in your QEMU installation:

```bash
qemu-system-x86_64 -device help 2>&1 | grep -i net
```

## VirtualBox Adapter Types

VirtualBox supports up to 8 virtual network adapters per VM (the GUI exposes 4; the remaining can be configured via `VBoxManage modifyvm`). Each adapter can emulate one of the following NIC models:

| Adapter                            | Chipset      | Speed     | Best For                                       |
|------------------------------------|--------------|-----------|-------------------------------------------------|
| AMD PCNet PCI II                   | `Am79C970A`  | 10 Mbps   | Legacy OSes (Windows 2000 and earlier)          |
| AMD PCNet FAST III                 | `Am79C973`   | 10/100 Mbps | Broadest guest OS compatibility; PXE/network boot via GRUB |
| Intel PRO/1000 MT Desktop          | `82540EM`    | 1 Gbps    | Modern Windows and Linux (default for most guests) |
| Intel PRO/1000 T Server            | `82543GC`    | 1 Gbps    | Server operating systems                        |
| Intel PRO/1000 MT Server           | `82545EM`    | 1 Gbps    | Server operating systems                        |
| Paravirtualized Network (virtio-net) | virtio     | —         | Maximum performance (Linux and modern Windows with virtio drivers) |

## Jumbo Frames

Standard Ethernet frames carry up to 1,500 bytes of payload (the MTU — Maximum Transmission Unit). **Jumbo frames** increase this to up to 9,000 bytes. By sending more data per frame, jumbo frames reduce the number of frames needed for large transfers, which lowers CPU overhead and improves throughput — particularly for bulk data workloads like storage, backups, and database replication.

Jumbo frames are disabled by default and must be enabled on every device in the path (NIC, bridge, switch, and guest interface). If any link in the chain has a smaller MTU, oversized frames are either fragmented or dropped.

**QEMU** — Jumbo frames are supported with `e1000`, `e1000e`, and `virtio-net-pci`. The `rtl8139` adapter does not support them (matching the real hardware's limitation). When using TAP networking, the host TAP interface, bridge, and physical NIC must all have their MTU set to 9000.

**VirtualBox** — Only Intel-based adapters (PRO/1000 family) support jumbo frames, and only in bridged networking mode. AMD PCNet adapters silently drop jumbo frames in both directions.

---

[← Previous: Host-Guest File Sharing](15_host_guest_file_sharing.md) | [Next: QEMU Networking Modes →](17_networking_modes_qemu.md)
