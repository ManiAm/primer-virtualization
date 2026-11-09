# Virtualization Primer

Virtualization has shaped computing since the 1960s, evolving from IBM mainframe time-sharing to today's microVMs and container orchestration platforms. It spans a broad spectrum — from hardware-level hypervisors that isolate full operating systems, to lightweight containers that package applications into portable units.

This primer covers the subject from the ground up: history and motivation, hypervisor types, hands-on emulation with QEMU, virtual networking, management tools (libvirt, Vagrant), containerization with Docker, lightweight virtualization (Firecracker, Kata Containers, gVisor), and network virtualization.

## Introduction & Hypervisors

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 01 | [Virtualization](docs/01_introduction.md)                             | History, motivation, and overview of virtualization          |
| 02 | [Hardware Virtualization](docs/02_hardware_virtualization.md)         | Hypervisors, Type 1 vs Type 2, consolidation, and HW extensions |
| 03 | [Type 1 Hypervisors](docs/03_type1_hypervisors.md)  | KVM architecture, Hyper-V, WSL, and compatibility           |

## QEMU Fundamentals

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 04 | [QEMU](docs/04_qemu.md)                                              | Quick Emulator overview, binary translation, and TCG         |
| 05 | [Virtual Disk Images](docs/05_virtual_disk_image.md)                  | Raw images, creation tools (`dd`, `truncate`, `qemu-img`)   |
| 06 | [QCOW2 Images](docs/06_qcow2_image.md)                               | QCOW2 format, clusters, snapshots, and internal structure   |
| 07 | [Minimal Root Filesystem](docs/07_minimal_root_fs.md)                 | Mounting QCOW2 images and creating a minimal root FS        |

## QEMU System & User Emulation

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 08 | [System Emulation (x86)](docs/08_qemu_system_emulation.md)            | Installing QEMU, emulating x86 VMs, serial redirection     |
| 09 | [System Emulation (ARM)](docs/09_qemu_system_emulation_arm.md)        | Cross-architecture emulation: ARM on x86 with pflash        |
| 10 | [Direct Linux Boot](docs/10_direct_linux_boot.md)                     | Booting kernels directly with and without initrd            |
| 11 | [QEMU and KVM](docs/11_qemu_and_kvm.md)                              | KVM acceleration and nested virtualization                  |
| 12 | [User-mode Emulation](docs/12_qemu_user_emulation.md)                | Running foreign-architecture binaries on the host           |

## QEMU Management & File Sharing

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 13 | [QEMU Monitor](docs/13_qemu_monitor.md)                              | Interactive monitor console for VM inspection and control   |
| 14 | [QEMU Machine Protocol (QMP)](docs/14_qemu_qmp.md)                   | JSON-based management protocol, Python client, QMP shell    |
| 15 | [Host-Guest File Sharing](docs/15_host_guest_file_sharing.md)        | USB passthrough, 9p/virtio-fs, ISO sharing, raw disk access |

## Virtual Networking

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 16 | [Virtual Network Adapters](docs/16_virtual_network_adapters.md)      | Virtual NICs and VirtualBox network adapter types            |
| 17 | [QEMU Networking Modes](docs/17_networking_modes_qemu.md)            | User-mode (SLiRP), TAP, bridge helper, socket, and VDE     |
| 18 | [VirtualBox Networking Modes](docs/18_networking_modes_virtualBox.md) | NAT, bridged, internal, host-only, and generic driver modes |

## VM Management Tools

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 19 | [libvirt](docs/19_libvirt.md)                                         | virsh, domains, networking, and VM lifecycle management     |
| 20 | [Vagrant](docs/20_vagrant.md)                                         | Vagrantfile, plugins, and reproducible VM environments      |

## Containers & OS-Level Virtualization

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 21 | [OS-Level Virtualization](docs/21_os_virtualization.md)               | Containerization concepts and container characteristics     |
| 22 | [Docker](docs/22_docker.md)                                          | Docker overview, components, daemon, and client-server model |
| 23 | [Docker Commands](docs/23_docker_commands.md)                        | Command reference for containers, images, networks, volumes |

## Docker Images & Dockerfiles

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 24 | [Docker Images](docs/24_docker_image.md)                              | Images, layers, registries, tags, and pull caching          |
| 25 | [Dockerfile Instructions](docs/25_dockerfile_instructions.md)        | `FROM`, `RUN`, `COPY`, `ADD`, `WORKDIR`, `SHELL`, and more |
| 26 | [Best Practices](docs/26_best_practice.md)                           | Layer immutability, multi-stage builds, and the `dive` tool |

## Docker Containers & Runtime

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 27 | [Docker Containers](docs/27_docker_container.md)                     | Interactive shells, main process, PID management, CMD       |
| 28 | [Environment Variables](docs/28_docker_env.md)                       | Passing, loading, and defaulting env vars; `ARG` vs `ENV`  |
| 29 | [Container Statistics](docs/29_docker_stat.md)                       | Resource monitoring and limiting container resources        |

## Docker Storage & Networking

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 30 | [Docker Volumes](docs/30_docker_volumes.md)                          | Bind mounts, named volumes, anonymous volumes, tmpfs        |
| 31 | [Docker Networking](docs/31_docker_network.md)                       | Bridge networks, DNS resolution, user-defined networks      |

## Docker Internals & Compose

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 32 | [Docker Internals](docs/32_docker_internals.md)                      | Evolution from LXC to containerd, OCI, and runC             |
| 33 | [Docker Compose](docs/33_docker_compose.md)                          | Multi-container orchestration, build, and environment config |

## Lightweight Virtualization

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 34 | [Lightweight Virtualization](docs/34_lightweight_virtualization.md)   | MicroVMs, Firecracker, Kata Containers, gVisor, unikernels  |

## Network Virtualization

| # | Topic | Description |
| -- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| 35 | [Network Virtualization](docs/35_network_virtualization.md)          | SDN, NFV, virtual switches, VLANs, overlays, and namespaces |
