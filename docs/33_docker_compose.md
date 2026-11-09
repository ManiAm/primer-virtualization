# Docker Compose

The previous chapters covered individual Docker concepts — images, containers, volumes, networks, and environment variables — using single `docker run` commands. In practice, most applications require **multiple containers** working together (a web server, a database, a cache). Managing each container manually becomes tedious and error-prone.

**Docker Compose** solves this by letting you define all your services, networks, and volumes in a single YAML file and manage them with one command.

## Why Compose?

Suppose you need an Nginx web server and a MySQL database, connected on the same network with persistent storage for the database.

**Without Compose** — five separate commands:

```bash
docker network create my-net
docker volume create mysql-data
docker volume create shared-data
docker run -d --name my-mysql --network my-net \
  -v mysql-data:/var/lib/mysql \
  -v shared-data:/shared \
  -e MYSQL_ROOT_PASSWORD=rootpassword mysql:latest
docker run -d --name my-nginx --network my-net \
  -v shared-data:/usr/share/nginx/html \
  -p 8080:80 nginx:latest
```

Every flag must be remembered and typed correctly. Stopping means running `docker stop` and `docker rm` for each container, then cleaning up the network and volumes.

**With Compose** — one file, one command:

```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:latest
    container_name: my-mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - mysql-data:/var/lib/mysql
      - shared-data:/shared
    networks:
      - my-net

  nginx:
    image: nginx:latest
    container_name: my-nginx
    ports:
      - "8080:80"
    volumes:
      - shared-data:/usr/share/nginx/html
    networks:
      - my-net

volumes:
  mysql-data:
  shared-data:

networks:
  my-net:
    driver: bridge
```

```bash
docker compose up -d
```

One command creates the network, the volumes, and both containers. The YAML file serves as **documentation** — anyone can read it and understand the entire application stack.

## Compose File Structure

A Compose file has three top-level keys:

| Key | Purpose |
|-----|---------|
| `services` | The containers to run (each service becomes one container) |
| `volumes`  | Named volumes shared across services (see [Chapter 30](30_docker_volumes.md)) |
| `networks` | Custom networks for service communication (see [Chapter 31](31_docker_network.md)) |

### Service Configuration

Each entry under `services` supports the same options you would pass to `docker run`:

| Key | Equivalent `docker run` Flag | Example |
|-----|------|---------|
| `image` | Image name | `nginx:latest` |
| `container_name` | `--name` | `my-nginx` |
| `ports` | `-p` | `"8080:80"` |
| `volumes` | `-v` | `mysql-data:/var/lib/mysql` |
| `environment` | `-e` | `MYSQL_ROOT_PASSWORD: secret` |
| `env_file` | `--env-file` | `- .db.env` |
| `networks` | `--network` | `- my-net` |
| `restart` | `--restart` | `always` |

### Service Dependencies (`depends_on`)

If one service needs another to start first, use `depends_on`:

```yaml
services:
  db:
    image: mysql:latest
    environment:
      MYSQL_ROOT_PASSWORD: secret

  web:
    image: nginx:latest
    depends_on:
      - db
```

Compose starts `db` before `web`. Note that `depends_on` only waits for the container to **start** — it does not wait for the application inside to be ready (e.g., MySQL accepting connections). For readiness checks, use the `healthcheck` option or application-level retry logic.

## Essential Commands

Navigate to the directory containing your `docker-compose.yml`, then:

| Command | Description |
|---------|-------------|
| `docker compose up -d` | Create and start all services in the background |
| `docker compose ps` | List running Compose services |
| `docker compose logs` | View combined logs from all services |
| `docker compose logs -f web` | Follow logs for a specific service |
| `docker compose stop` | Stop all services (containers remain) |
| `docker compose down` | Stop and remove containers and networks |
| `docker compose down -v` | Same as above, **plus** remove named volumes |

The difference between `stop` and `down` matters:

- `stop` — containers are stopped but preserved (can be restarted with `docker compose start`)
- `down` — containers and networks are **removed** (a clean teardown)
- `down -v` — also removes the named volumes (**deletes persistent data**)

## Building Images with Compose

Instead of pulling a pre-built image, Compose can build from a Dockerfile:

```yaml
services:
  myapp:
    build: .
    image: myapp:latest
    ports:
      - "3000:3000"
```

| Key | Meaning |
|-----|---------|
| `build: .` | Build from the Dockerfile in the current directory |
| `image: myapp:latest` | Tag the built image with this name |

Useful build commands:

| Command | Description |
|---------|-------------|
| `docker compose build` | Build (or rebuild) all images |
| `docker compose up --build` | Build images **and** start services in one step |

Compose only rebuilds if it detects changes in the Dockerfile or build context. To force a full rebuild:

```bash
docker compose build --no-cache
```

## Environment Variables in Compose

Compose supports the same environment variable mechanisms covered in [Chapter 28](28_docker_env.md) — inline values, env files, and Dockerfile defaults — plus one Compose-specific feature: **YAML interpolation**.

### Inline and Env File

```yaml
services:
  web:
    image: alpine:3
    environment:
      APP_NAME: demo        # inline value
      PORT: 8080
    env_file:
      - .db.env             # bulk import from file
```

### YAML Interpolation with `${VAR}`

Compose substitutes `${VAR}` references in the YAML **before** sending the configuration to Docker. You can set defaults with `:-` syntax:

```yaml
services:
  web:
    image: ${IMAGE_NAME:-nginx}:${IMAGE_TAG:-latest}
    ports:
      - "${HOST_PORT:-8080}:80"
```

Variable values come from:
1. **Shell environment** — variables exported in your terminal
2. **`.env` file** — a file named `.env` in the same directory as the Compose file (loaded automatically)

Example `.env` file:

```
IMAGE_NAME=nginx
IMAGE_TAG=1.25
HOST_PORT=9090
```

### The `$$` Escape

Because `$` triggers Compose interpolation, passing a literal `$` into a container command requires escaping it as `$$`:

```yaml
services:
  web:
    image: alpine:3
    environment:
      APP_NAME: demo
    command: sh -c 'echo "APP_NAME=$$APP_NAME"; sleep 3600'
```

Compose turns `$$` into `$` before sending the command to Docker, so the container's shell expands `$APP_NAME` at runtime. Without the escape, Compose would try to substitute `${APP_NAME}` from the YAML context — which may produce a different (or empty) value.

---

[← Previous: Docker Internals](32_docker_internals.md) | [Next: Lightweight Virtualization →](34_lightweight_virtualization.md)
