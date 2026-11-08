# Container Statistics and Resource Limits

This chapter covers **monitoring** container resource usage and **limiting** the resources a container can consume.

## Monitoring with `docker stats`

The `docker stats` command displays a **live, auto-updating** stream of resource usage for all running containers:

```bash
docker stats
```

```
CONTAINER ID   NAME       CPU %   MEM USAGE / LIMIT   MEM %   NET I/O         BLOCK I/O     PIDS
a1b2c3d4e5f6   my-nginx   0.02%   3.5MiB / 7.7GiB    0.04%   1.2kB / 648B    0B / 4.1kB    3
f6e5d4c3b2a1   my-redis   0.15%   8.1MiB / 7.7GiB    0.10%   2.4kB / 1.1kB   0B / 0B       5
```

| Column        | Description |
| ------------- | --- |
| **CPU %**     | Percentage of the host's CPU used by this container |
| **MEM USAGE / LIMIT** | Current memory usage and the container's memory limit (or total host RAM if unlimited) |
| **MEM %**     | Memory usage as a percentage of the limit |
| **NET I/O**   | Total bytes received and sent over the network |
| **BLOCK I/O** | Total bytes read from and written to disk |
| **PIDs**      | Number of processes or threads running inside the container |

### Monitor Specific Containers

Pass one or more container names or IDs to monitor only those:

```bash
docker stats my-nginx my-redis
```

### Single Snapshot (Non-Streaming)

By default, `docker stats` refreshes continuously. Use `--no-stream` for a **one-time snapshot**, which is useful in scripts or CI pipelines:

```bash
docker stats --no-stream
```

### Custom Output Format

Use `--format` to select specific columns:

```bash
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

```
NAME       CPU %   MEM USAGE / LIMIT
my-nginx   0.02%   3.5MiB / 7.7GiB
my-redis   0.15%   8.1MiB / 7.7GiB
```

## Limiting Container Resources

By default, a container has **no resource limits** — it can use as much CPU, memory, and I/O as the host allows. This is risky in multi-container environments because a single container can starve others of resources or even crash the host.

Docker uses Linux **cgroups** (control groups) under the hood to enforce resource limits. You set these limits with flags on `docker run`, or update them on an **already running** container with `docker update`:

```bash
docker update --memory=1g --cpus=2.0 my-app
```

### Memory Limits

```bash
docker run -d --memory=512m --name my-app my-image
```

| Flag | Description |
| --- | --- |
| `--memory` (or `-m`) | Maximum memory the container can use (e.g., `256m`, `1g`) |
| `--memory-swap` | Total memory + swap limit. Set equal to `--memory` to disable swap |
| `--memory-reservation` | Soft limit — Docker tries to keep usage below this, but allows bursts |

If a container exceeds its memory limit, the kernel's **OOM (Out of Memory) killer** terminates it. You can see this in `docker inspect`:

```bash
docker inspect --format='{{.State.OOMKilled}}' my-app
```

### CPU Limits

```bash
docker run -d --cpus=1.5 --name my-app my-image
```

| Flag | Description |
| --- | --- |
| `--cpus` | Maximum number of CPU cores (e.g., `1.5` = one and a half cores) |
| `--cpu-shares` | Relative CPU weight (default: 1024). Only matters when containers compete for CPU |
| `--cpuset-cpus` | Pin the container to specific CPU cores (e.g., `0,2` or `0-3`) |

`--cpus` is a **hard limit** — the container cannot exceed it. `--cpu-shares` is a **relative weight** — it only takes effect when multiple containers compete for CPU time. A container with 2048 shares gets twice the CPU time of one with 1024 shares.

### Combining Limits

In practice, set both memory and CPU limits together:

```bash
docker run -d \
  --memory=512m \
  --memory-swap=512m \
  --cpus=1.0 \
  --name my-app my-image
```

This gives the container at most 512 MB of memory (no swap) and one CPU core — preventing it from monopolizing host resources.

---

[← Previous: Environment Variables](28_docker_env.md) | [Next: Docker Volumes →](30_docker_volumes.md)
