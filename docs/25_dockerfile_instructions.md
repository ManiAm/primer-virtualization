# Dockerfile Instructions

A Dockerfile is a text file containing instructions that Docker executes sequentially to build an image. Each instruction creates a layer in the resulting image. The [previous chapter](24_docker_image.md) covered images, layers, and caching — this chapter details every common Dockerfile instruction.

> **Note:** Instructions are not case-sensitive, but by convention they are written in UPPERCASE to distinguish them from their arguments.

## Instruction Summary

| Instruction | Description |
| --- | --- |
| `FROM` | Set the base image |
| `WORKDIR` | Set the working directory inside the container |
| `RUN` | Execute a command during the build (creates a new layer) |
| `SHELL` | Change the default shell for `RUN` commands |
| `COPY` | Copy files from the build context into the image |
| `ADD` | Like `COPY`, but also handles URLs and auto-extracts archives |
| `LABEL` | Add key-value metadata to the image |
| `ENV` | Set environment variables (persist into running containers) |
| `ARG` | Define build-time variables (not available at runtime) |
| `USER` | Set the user for subsequent instructions and the running container |
| `EXPOSE` | Document which ports the container listens on (does not publish them) |
| `CMD` | Set the default command when the container starts |
| `ENTRYPOINT` | Set a fixed command that always runs (not easily overridden) |
| `HEALTHCHECK` | Define a command to check container health |
| `STOPSIGNAL` | Set the signal used to stop the container |
| `VOLUME` | Create a mount point for persistent storage |
| `ONBUILD` | Add a trigger instruction for downstream builds |

## FROM

Every Dockerfile must begin with a `FROM` instruction. It sets the **base image** — the starting point for all subsequent layers.

```dockerfile
FROM ubuntu:24.04
FROM python:3.12-slim
FROM node:20-alpine
```

By default, `docker build` uses the locally cached version of the base image and does **not** check the registry for updates. To force Docker to pull the latest version:

```bash
docker build --pull --tag my-image .
```

## WORKDIR

Sets the working directory inside the container. All subsequent `RUN`, `CMD`, `COPY`, and `ADD` instructions operate relative to this path. If the directory doesn't exist, Docker creates it.

```dockerfile
WORKDIR /app
```

Without `WORKDIR`, the default is `/` (root). Setting an explicit working directory makes the Dockerfile easier to read and avoids path errors.

## RUN

Executes a command at **build time** and commits the result as a new image layer. Common uses include installing packages, building code, and configuring the environment.

```dockerfile
RUN apt-get update && apt-get install -y curl
RUN npm install
RUN mkdir -p /app/logs && chown -R www-data:www-data /app
```

By default, `RUN` uses `/bin/sh -c` on Linux and `cmd /S /C` on Windows. Use the `SHELL` instruction to change this.

> **Tip:** Combine related commands with `&&` in a single `RUN` instruction to reduce the number of layers.

## SHELL

Changes the default shell used by `RUN` instructions. Useful for switching to `bash` (for features like arrays and better error handling) or `powershell` on Windows.

```dockerfile
FROM ubuntu:latest
SHELL ["/bin/bash", "-c"]
RUN echo "Using Bash instead of sh"
```

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022
SHELL ["powershell", "-Command"]
RUN Write-Host "Running PowerShell commands in Dockerfile"
```

## COPY

Copies files and directories from the **build context** (the directory passed to `docker build`) into the image.

```dockerfile
COPY index.html /usr/share/nginx/html/
COPY src/ /app/src/
```

## ADD

Similar to `COPY`, but with two additional capabilities:

```dockerfile
ADD archive.tar.gz /app/            # auto-extracts tar archives
ADD https://example.com/file.txt /app/   # fetches from URLs
```

> **Best practice:** Use `COPY` unless you specifically need auto-extraction or URL fetching. `COPY` is more transparent and predictable.

## LABEL

Adds key-value metadata to the image. Useful for version tracking, maintainer info, and automation.

```dockerfile
LABEL maintainer="Jane Doe <jane@example.com>"
LABEL version="2.0"
LABEL description="Production web server image"
```

View labels with `docker inspect`:

```bash
docker inspect --format='{{json .Config.Labels}}' my-image
```

## ENV and ARG

Docker provides two ways to parameterize builds and containers. `ENV` sets **runtime** variables; `ARG` sets **build-time** variables.

### ENV

Sets environment variables that persist across the build **and** into running containers. Any process inside the container can access them.

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
```

Override at runtime with the `-e` flag:

```bash
docker run -e NODE_ENV=development -e PORT=8080 my-app
```

> **ENV vs RUN export:** `ENV` variables persist for the lifetime of the container. `RUN export VAR=value` sets a variable only for that single `RUN` instruction — it does not persist to subsequent layers or the running container.

### ARG

Defines build-time variables that can be passed with `--build-arg`.

A common use case is **pinning the base image version** so you can change it from the CLI without editing the Dockerfile:

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "index.js"]
```

```bash
docker build --tag my-app .                              # uses Node 20
docker build --build-arg NODE_VERSION=18 --tag my-app .  # uses Node 18
```

> **Note:** `ARG` declared **before** `FROM` is only available in the `FROM` line itself. To use it in later instructions, declare it again after `FROM`.

Another common use is **embedding version metadata** into the image:

```dockerfile
ARG APP_VERSION=1.0
LABEL version=$APP_VERSION
```

```bash
docker build --build-arg APP_VERSION=2.0 --tag my-app .
```

Unlike `ENV`, `ARG` values are **not** available in the running container — they only exist during the build.

### Comparison

|                            | `ARG`                     | `ENV` |
| -------------------------- | ------------------------- | --- |
| **Available during build** | ✅ Yes                    | ✅ Yes |
| **Available at runtime**   | ❌ No                     | ✅ Yes |
| **Set from CLI**           | `--build-arg`             | `-e` or `--env` |
| **Visible in final image** | No (but in build history) | Yes |

### `ARG` → `ENV` Pattern

A common pattern is to accept a value at build time with `ARG` and copy it into `ENV` for runtime use:

```dockerfile
FROM alpine:3
ARG API_BASE=https://api.example.com
ENV API_URL=${API_BASE}/v1
CMD ["sh", "-c", "echo API_URL=$API_URL"]
```

```bash
docker build --build-arg API_BASE=https://api.dev -t my-app .
docker run --rm my-app
# API_URL=https://api.dev/v1
```

> **Security note:** Both `ARG` and `ENV` values are visible in the image metadata (`docker history`, `docker inspect`). Never store secrets (passwords, API keys, tokens) in either. Use Docker secrets, mounted files, or runtime-only `-e` flags sourced from a secrets manager instead.

Runtime usage of environment variables is covered in [Chapter 28 — Environment Variables](28_docker_env.md).

## USER

Sets the user (and optionally group) for subsequent `RUN`, `CMD`, and `ENTRYPOINT` instructions, and for the running container.

```dockerfile
RUN useradd -m appuser
USER appuser
```

Running as a non-root user is a security best practice — it limits the damage if a container is compromised. The user must already exist in the image; create it with `RUN` before the `USER` instruction.

## EXPOSE

Documents which port(s) the container listens on at runtime. This is **informational only** — it does not publish the port.

```dockerfile
EXPOSE 80
EXPOSE 443
```

To actually make the port accessible from the host, use `-p` with `docker run`:

```bash
docker run -d -p 8080:80 my-image    # maps host:8080 → container:80
```

## CMD and ENTRYPOINT

Both instructions define what runs when a container starts. `CMD` sets a **default** that can be easily overridden; `ENTRYPOINT` sets a **fixed** command that always runs.

### CMD

Sets the **default command** that runs when the container starts. It has two forms:

| Form | Syntax | Shell Involved? |
| --- | --- | --- |
| Exec (preferred) | `CMD ["executable", "arg1", "arg2"]` | No — runs the executable directly |
| Shell | `CMD command arg1 arg2` | Yes — runs via `/bin/sh -c` |

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

`CMD` can be **overridden** at runtime by appending a command to `docker run`:

```bash
docker run my-image /bin/bash    # overrides CMD
```

> **Important:** The `CMD` process should be **blocking** (long-running). Docker ties the container lifecycle to its main process (PID 1) — when the process exits, the container stops. A non-blocking command like `CMD ["echo", "hello"]` causes the container to exit immediately.

### ENTRYPOINT

Sets a **fixed command** that always executes when the container starts. Unlike `CMD`, it is not easily overridden.

| Form | Syntax |
| --- | --- |
| Exec (preferred) | `ENTRYPOINT ["executable", "arg1"]` |
| Shell | `ENTRYPOINT command arg1` |

### Combining ENTRYPOINT and CMD

`ENTRYPOINT` and `CMD` are often combined — `ENTRYPOINT` defines the command and `CMD` provides default arguments:

```dockerfile
ENTRYPOINT ["top"]
CMD ["-b"]
```

The effective command is `top -b`. At runtime, you can override the `CMD` portion:

```bash
docker run my-image -n 5    # runs: top -n 5
```

## HEALTHCHECK

Defines a command that Docker runs periodically to check whether the container is healthy. If the check fails repeatedly, Docker marks the container as `unhealthy`.

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

| Option | Default | Description |
| --- | --- | --- |
| `--interval` | 30s | Time between health checks |
| `--timeout` | 30s | Maximum time for the check to complete |
| `--retries` | 3 | Consecutive failures before marking unhealthy |
| `--start-period` | 0s | Grace period before the first check |

Check container health:

```bash
docker inspect --format='{{json .State.Health}}' my-container
```

## STOPSIGNAL

Changes the signal Docker sends when stopping the container. The default is `SIGTERM`.

```dockerfile
STOPSIGNAL SIGINT
```

When you run `docker stop`, Docker sends this signal first, waits up to 10 seconds (configurable with `-t`), then sends `SIGKILL` if the container hasn't exited.

## VOLUME

Creates a mount point and marks it as holding persistent data. Data written to this path is stored outside the container's writable layer and persists even after the container is removed.

```dockerfile
VOLUME /var/lib/mysql
```

At runtime, Docker creates an anonymous volume for this path. You can override it with a named volume or bind mount using `-v`:

```bash
docker run -v my-data:/var/lib/mysql my-image
```

Volumes are covered in detail in [Chapter 30 — Docker Volumes](30_docker_volumes.md).

## ONBUILD

Adds a trigger instruction that executes when the image is used as a `FROM` base in another Dockerfile. Useful for library or framework images that need downstream builds to run specific steps.

```dockerfile
# In the base image Dockerfile
ONBUILD COPY . /app
ONBUILD RUN npm install
```

When another Dockerfile uses `FROM my-base-image`, the `ONBUILD` instructions execute automatically during that downstream build.

---

[← Previous: Docker Images](24_docker_image.md) | [Next: Best Practices →](26_best_practice.md)
