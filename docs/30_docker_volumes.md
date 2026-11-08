# Docker Volumes

This chapter covers **volumes** — Docker's mechanism for persisting data beyond the container lifecycle.

## Why Volumes?

As covered in [Chapter 27](27_docker_container.md#what-happens-to-data-in-each-state), changes made inside a container are stored in its writable layer. When the container is **removed**, that layer — and all its data — is deleted. Volumes solve this by storing data **outside** the container's writable layer, on the host filesystem or in memory.

Volumes also enable **sharing data** between multiple containers — useful for applications that require shared state, logs, or configuration.

## Volume Types

Docker supports four types of storage mounts:

| Type                 | Data Location                 | Persists After Container Removal? | Use Case |
| -------------------- | ----------------------------- | --- | --- |
| **Bind mount**       | Specific host path you choose | ✅ Yes (on host) | Development, config files |
| **Named volume**     | Docker-managed directory      | ✅ Yes | Databases, shared data |
| **Anonymous volume** | Docker-managed directory (random name) | ✅ Yes (until pruned) | Dockerfile `VOLUME` instruction |
| **tmpfs mount**      | Host RAM only                 | ❌ No (memory only) | Sensitive data, temp caches |

## Bind Mounts

A bind mount maps a specific file or directory on the **host** to a path inside the container:

```bash
docker run -v /host/data:/container/data nginx
```

Changes are **bidirectional** — if you modify files on the host, the changes are immediately visible inside the container, and vice versa. This makes bind mounts ideal for development workflows where you want to edit source code on the host and see changes reflected in the container without rebuilding.

> **Note:** The host path must be an **absolute path**. If the host directory doesn't exist, Docker creates it automatically.

## Named Volumes

Named volumes are created and managed by Docker. They are identified by a human-readable name and can be shared across multiple containers.

Create a named volume:

```bash
docker volume create my_volume
```

Mount it in a container:

```bash
docker run --name container1 -v my_volume:/data -d nginx
```

Mount the **same volume** in another container to share data:

```bash
docker run --name container2 -v my_volume:/data -d nginx
```

Both containers read from and write to the same `/data` directory. Named volumes persist independently of any container — they are **not** deleted when the containers using them are removed. You must delete them explicitly:

```bash
docker volume rm my_volume
```

## Anonymous Volumes

Anonymous volumes are created when you specify a container path without a host path or volume name:

```bash
docker run -v /data nginx
```

Docker creates a volume with a random hash name (e.g., `a1b2c3d4e5f6...`). Anonymous volumes are also created automatically by the `VOLUME` instruction in a Dockerfile (see [Chapter 25 — VOLUME](25_dockerfile_instructions.md#volume)).

**Anonymous volumes persist after the container is removed** — they are not automatically deleted. They accumulate as dangling volumes over time. The only exception is when you start the container with `--rm`, in which case Docker removes the anonymous volume when the container exits:

```bash
docker run --rm -v /data nginx    # volume is deleted when container exits
```

Because anonymous volumes have random names, they are difficult to identify and manage. **Prefer named volumes** for any data you care about.

## tmpfs Mounts

A tmpfs mount stores data in the **host's RAM** — nothing is written to disk:

```bash
docker run --tmpfs /container/data nginx
```

tmpfs mounts are:

- **Fast** — RAM is much faster than disk I/O
- **Ephemeral** — Data does not persist across container restarts
- **Secure** — Sensitive data (tokens, temp keys) is never written to disk

## Managing Volumes

```bash
docker volume ls                        # list all named and anonymous volumes
docker volume inspect my_volume         # show volume details (mount point, driver)
docker volume rm my_volume              # remove a specific volume
docker volume ls -f dangling=true       # list volumes not attached to any container
docker volume prune                     # remove all unused volumes
```

> **Note:** Bind mounts do not appear in `docker volume ls` — only named and anonymous volumes are listed.

### Volume Location

Docker stores named and anonymous volumes in its managed storage directory. Use `docker volume inspect` to find the exact path:

```bash
docker volume inspect my_volume
```

```json
[
    {
        "CreatedAt": "2025-02-15T12:46:07-08:00",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/my_volume/_data",
        "Name": "my_volume",
        "Scope": "local"
    }
]
```

The `Mountpoint` is the directory on the host where Docker stores the volume data. You need root privileges (`sudo`) to access it directly.

## How Volumes Overlay Container Files

When a volume is mounted to a path inside a container, it **overlays** whatever was there originally. The container's original files at that path become hidden (not deleted) for the duration of the mount.

- If the mount is a **directory**, all files and subdirectories at that path are hidden
- If the mount is a **single file**, only that specific file is replaced

When the container stops and the volume is unmounted, the original files become visible again.

### Example: Replacing Nginx's Default Page

Nginx serves content from `/usr/share/nginx/html/index.html`. Start a default container:

```bash
docker run --name nginx-default -d -p 8080:80 nginx
```

Visit `http://localhost:8080` — you see the default "Welcome to Nginx!" page.

Now create a custom page on the host:

```bash
mkdir ~/nginx-html
echo "<h1>Custom Nginx Page</h1>" > ~/nginx-html/index.html
```

Start a new container with the custom file mounted:

```bash
docker run --name nginx-custom \
    -v ~/nginx-html/index.html:/usr/share/nginx/html/index.html \
    -d -p 8081:80 nginx
```

Visit `http://localhost:8081` — you see "Custom Nginx Page". The original `index.html` is hidden by the bind mount but still exists in the image layer.

---

[← Previous: Container Statistics](29_docker_stat.md) | [Next: Docker Networking →](31_docker_network.md)
