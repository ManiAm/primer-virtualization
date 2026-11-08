# Network Virtualization

The [introduction](01_introduction.md) identified three main categories of virtualization: hardware, operating system, and network. The primer covers hardware virtualization in depth ([Chapters 2–20](02_hardware_virtualization.md)) and OS-level virtualization through Docker ([Chapters 21–33](21_os_virtualization.md)). This chapter introduces the third category — **network virtualization** — and connects it to the networking concepts that appeared throughout the primer.

## What is Network Virtualization?

Network virtualization is an umbrella term for technologies that recreate network infrastructure in software. Just as a hypervisor lets multiple VMs share one physical server, network virtualization lets multiple isolated virtual networks share the same physical network hardware.

The umbrella covers several distinct technologies — virtual switches, VLANs, overlay networks, network namespaces, SDN, and NFV — each solving a different piece of the puzzle. The sections below introduce them one at a time.

## Key Concepts

### Virtual Switches

A **virtual switch** is a software program running on a host that connects virtual machines, containers, and physical network interfaces. It performs the same job as a physical Ethernet switch — learning MAC addresses and forwarding frames — but entirely in software.

We have used virtual switches throughout this primer without always calling them that. The Linux bridge `virbr0` serves as the default NAT bridge for KVM virtual machines ([Chapter 19](19_libvirt.md)), `docker0` serves the same role for Docker containers ([Chapter 31](31_docker_network.md)), and QEMU's bridge helper connects TAP devices to a host bridge ([Chapter 17](17_networking_modes_qemu.md)).

**Open vSwitch (OVS)** is the most widely deployed production virtual switch. It supports advanced features like VLAN tagging, traffic mirroring, flow-based forwarding, and integration with SDN controllers (described below).

### VLANs (Virtual LANs)

A **VLAN** segments a single physical switch into multiple isolated groups. Normally, all devices on the same switch can see each other's traffic (this shared space is called a **broadcast domain**). VLANs create separate broadcast domains so that devices on one VLAN cannot see traffic from another — even if they are plugged into the same physical switch.

VLANs are identified by a numeric tag (1–4094) added to Ethernet frames per the **IEEE 802.1Q** standard. A switch port is configured to carry one or more VLAN tags, and traffic from different VLANs is kept separate.

```
Physical Switch
┌────────────────────────────────┐
│  VLAN 10 (Engineering)         │
│    Port 1 ── Port 3 ── Port 7  │
│                                │
│  VLAN 20 (Marketing)           │
│    Port 2 ── Port 5 ── Port 8  │
└────────────────────────────────┘
```

Devices on VLAN 10 cannot reach devices on VLAN 20 without a router — even though they share the same physical switch.

### Overlay Networks

An **overlay network** is a virtual network built on top of an existing physical (underlay) network. Overlay packets are **encapsulated** inside regular IP packets, allowing virtual networks to span multiple physical hosts, data centers, or even cloud regions.

Common overlay technologies:

| Technology | Encapsulation | Use Case |
|------------|---------------|----------|
| **VXLAN**  | UDP (port 4789) | Data center overlay networks; supports ~16 million segments |
| **GRE**    | IP protocol 47 | Point-to-point tunnels between routers or hosts |
| **Geneve** | UDP (port 6081) | Next-generation overlay; extensible header format |

We saw overlay networks in practice in [Docker networking](31_docker_network.md): Docker's `overlay` driver uses VXLAN to connect containers across multiple Docker hosts, enabling multi-host communication in Docker Swarm clusters.

### Network Namespaces

A **network namespace** is a Linux kernel feature that gives a process (or group of processes) its own isolated network stack — its own interfaces, routing table, iptables rules, and ARP table. From inside a namespace, only the interfaces assigned to it are visible.

Network namespaces are the foundation of container networking:

- Each Docker container runs in its own network namespace
- A **veth pair** (virtual Ethernet cable) connects the container's namespace to the host bridge (`docker0`)
- The host bridge connects containers to each other and to the outside world via NAT

This is the same pattern described in [Docker networking](31_docker_network.md) and mirrors how [libvirt](19_libvirt.md) connects VMs using TAP devices and `virbr0`.

### Software-Defined Networking (SDN)

**SDN** is an architecture that separates the **control plane** (which decides where traffic goes) from the **data plane** (which forwards packets). In a traditional network, every switch and router runs its own control logic. In SDN, a centralized **controller** makes routing decisions and programs the switches via a standard protocol.

```
┌─────────────────────────┐
│     SDN Controller      │  ← Control Plane (routing decisions)
│   (e.g., OpenDaylight)  │
└────────┬────────────────┘
         │ OpenFlow protocol
    ┌────┴────┬───────────┐
    ▼         ▼           ▼
┌───────┐ ┌───────┐ ┌───────┐
│Switch │ │Switch │ │Switch │  ← Data Plane (packet forwarding)
└───────┘ └───────┘ └───────┘
```

Benefits of SDN:

- **Centralized management** — one controller configures the entire network
- **Programmability** — network behavior can be changed through software APIs
- **Automation** — new networks can be provisioned in seconds, not days
- **Vendor independence** — switches from different vendors speak the same OpenFlow protocol

### Network Functions Virtualization (NFV)

In a traditional network, every function is a dedicated physical appliance — a hardware firewall, a hardware load balancer, a hardware router. Each must be purchased, installed, cabled, and configured individually. This makes provisioning slow, scaling expensive, and testing difficult.

**NFV** replaces these dedicated appliances with software running on standard servers:

| Physical Appliance | NFV Equivalent |
|--------------------|----------------|
| Hardware firewall  | Virtual firewall (e.g., pfSense VM) |
| Hardware load balancer | Software load balancer (e.g., HAProxy, Nginx) |
| Hardware router    | Software router (e.g., VyOS, FRRouting) |
| WAN optimizer      | Virtual WAN optimizer |

Because these functions are now software, they can be deployed, scaled, and updated as easily as any other application — no truck rolls or change-management windows required.

NFV and SDN are complementary: SDN manages the network fabric, while NFV provides the network services that run on top of it.

## Connection to This Primer

Many networking concepts covered earlier in the primer are practical applications of network virtualization:

| Primer Topic | Network Virtualization Concept |
|-------------|-------------------------------|
| Virtual NICs ([Chapter 16](16_virtual_network_adapters.md)) | Emulated and paravirtualized network interfaces |
| QEMU TAP/bridge networking ([Chapter 17](17_networking_modes_qemu.md)) | Virtual switches, veth-like connectivity |
| VirtualBox network modes ([Chapter 18](18_networking_modes_virtualBox.md)) | NAT, bridged, and internal virtual networks |
| libvirt `virbr0` and dnsmasq ([Chapter 19](19_libvirt.md)) | Virtual switch with built-in DHCP/DNS |
| Docker `docker0` bridge ([Chapter 31](31_docker_network.md)) | Virtual switch, network namespaces, veth pairs |
| Docker overlay driver ([Chapter 31](31_docker_network.md)) | VXLAN overlay networks |

These tools demonstrate network virtualization at a small scale — a single host or a handful of containers. In production data centers and cloud platforms, the same concepts (virtual switches, overlays, SDN controllers, NFV) operate at the scale of thousands of hosts and millions of network flows.

## Quick Reference

| Concept | One-Line Summary |
|---------|-----------------|
| Virtual Switch | Software switch connecting VMs/containers to the network |
| VLAN | Logical segmentation of a physical network using 802.1Q tags |
| Overlay Network | Virtual network tunneled over an existing physical network |
| Network Namespace | Linux kernel isolation of an entire network stack |
| SDN | Centralized control plane separated from distributed data plane |
| NFV | Network appliances replaced by software on standard servers |

---

[← Previous: Docker Compose](33_docker_compose.md)
