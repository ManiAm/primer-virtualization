# OS-Level Virtualization

The previous chapters covered hardware virtualization — hypervisors, QEMU, KVM, virtual networking, and management tools like libvirt and Vagrant. All of these technologies run full virtual machines, each with its own guest OS kernel. This chapter introduces a fundamentally different approach: **OS-level virtualization** (commonly called **containerization**), where multiple isolated environments share a single host kernel.

## VMs vs. Containers

The diagram below illustrates the key architectural difference:

<img src="../pics/os_virt.jpg" alt="VMs vs Containers" width="800">

**Virtual machines** each include a full guest OS (kernel + userspace) on top of a hypervisor. This provides strong isolation but adds significant overhead — every VM carries its own kernel, drivers, and system libraries.

**Containers** share the host OS kernel and isolate applications at the process level using kernel features like namespaces and cgroups. Each container packages only the application and its dependencies — no separate kernel is needed. This makes containers far lighter than VMs.

## Container Characteristics

| Characteristic | Description |
| --- | --- |
| **Portability** | A container bundles the application with its dependencies, libraries, and configuration. It runs the same way on a developer's laptop, a data center server, or the cloud — solving the classic "it works on my machine" problem. |
| **Lightweight** | Containers share the host kernel and isolate at the process level, requiring far less overhead than VMs (which virtualize the entire hardware stack). |
| **Fast Startup** | Containers start in seconds (vs. minutes for VMs), enabling rapid deployment and scaling. |
| **Efficiency & Scalability** | Because they are lightweight and fast, containers make it easy to scale applications up or down by adding or removing instances in response to demand. |
| **Microservices** | Containerization is well-suited to microservices architecture, where an application is built as a collection of small, independent services. Each service runs in its own container and can be deployed, scaled, and updated independently. |

## When to Use VMs vs. Containers

| Criteria | Virtual Machines | Containers |
| --- | --- | --- |
| Isolation level | Hardware-level (separate kernel per VM) | Process-level (shared host kernel) |
| Overhead | High — full OS per VM | Low — shared kernel, minimal footprint |
| Startup time | Minutes | Seconds |
| Best for | Running different OSes, strong security isolation, compliance requirements | Rapid deployment, scaling, portability, CI/CD pipelines |
| Example | Running Windows and Linux side by side on the same host | Deploying 50 instances of a web service behind a load balancer |

VMs and containers are not mutually exclusive — in practice, containers often run inside VMs to combine the security isolation of hardware virtualization with the efficiency of containerization.

## Containerization Timeline

Containerization is not a new concept. Various technologies have evolved over two decades:

| Technology | Year | Description |
| --- | --- | --- |
| HP-UX Secure Resource Partitions (SRP) | 2002 | Early OS-level isolation on HP-UX Unix systems |
| Solaris Containers (Zones) | 2004 | Lightweight virtualization built into Sun's Solaris OS |
| OpenVZ | 2005 | Linux container technology using a patched kernel |
| AIX Workload Partitions (WPARs) | 2007 | OS-level virtualization on IBM's AIX Unix |
| LXC (Linux Containers) | 2008 | First Linux container implementation using cgroups and namespaces |
| Docker | 2013 | Made containers accessible with a simple CLI, image format, and ecosystem |
| Podman | 2018 | Daemonless, rootless alternative to Docker with a compatible CLI |

## Docker and Podman

**Docker** is the technology that popularized containerization. It provides a simple CLI, a standardized image format, and a rich ecosystem (Docker Hub, Docker Compose, Docker Desktop). Docker uses a client-server model where the `docker` CLI sends commands to a background daemon (`dockerd`) that manages containers. Docker is covered in detail in [Chapters 22–33](22_docker.md).

**Podman** offers a Docker-compatible CLI (`podman` commands mirror `docker` commands) but with a different architecture:

| Aspect | Docker | Podman |
| --- | --- | --- |
| Architecture | Client-server (requires `dockerd` daemon) | Daemonless (each command forks its own process) |
| Root privileges | Daemon runs as root by default | Rootless by default — no daemon, no root required |
| Security | Daemon compromise can affect the entire host | Reduced attack surface due to rootless, daemonless design |
| CLI compatibility | `docker run`, `docker ps`, etc. | `podman run`, `podman ps`, etc. (drop-in replacement) |

---

[← Previous: Vagrant](20_vagrant.md) | [Next: Docker →](22_docker.md)
