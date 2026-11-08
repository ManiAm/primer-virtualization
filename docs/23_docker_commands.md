# Docker Commands

Docker 1.13 reorganized the CLI into **management commands** grouped by object type. The new syntax follows the pattern:

```
docker <object> <command> [options]
```

For example, `docker run` became `docker container run`, and `docker images` became `docker image ls`. The old shorthand commands still work as permanent aliases, but the new syntax is recommended because it makes the object type explicit.

## Container Commands

| Shorthand | Full Command | Description |
| --- | --- | --- |
| `docker run` | `docker container run` | Create and start a new container from an image |
| `docker create` | `docker container create` | Create a container without starting it |
| `docker start` | `docker container start` | Start a stopped container |
| `docker stop` | `docker container stop` | Gracefully stop a running container (SIGTERM, then SIGKILL after 10s) |
| `docker restart` | `docker container restart` | Stop and start a container |
| `docker kill` | `docker container kill` | Immediately stop a container (sends SIGKILL) |
| `docker pause` | `docker container pause` | Freeze all processes in a container |
| `docker unpause` | `docker container unpause` | Resume a paused container |
| `docker rm` | `docker container rm` | Remove a stopped container (`-f` to force-remove a running one) |
| `docker ps` | `docker container ls` | List running containers (`-a` to include stopped) |
| `docker exec` | `docker container exec` | Run a command inside a running container |
| `docker attach` | `docker container attach` | Attach terminal to a running container's main process |
| `docker logs` | `docker container logs` | View a container's stdout/stderr output |
| `docker inspect` | `docker container inspect` | Show detailed JSON metadata for a container |
| `docker stats` | `docker container stats` | Live resource usage (CPU, memory, network, disk I/O) |
| `docker cp` | `docker container cp` | Copy files between a container and the host |
| `docker commit` | `docker container commit` | Create a new image from a container's current state |
| `docker rename` | `docker container rename` | Rename a container |
| `docker port` | `docker container port` | List port mappings for a container |
| `docker top` | `docker container top` | Display running processes inside a container |
| `docker export` | `docker container export` | Export a container's filesystem as a tar archive |
| `docker update` | `docker container update` | Update CPU, memory, or restart policy of a running container |
| `docker wait` | `docker container wait` | Block until a container stops, then print its exit code |
| `docker diff` | `docker container diff` | Show filesystem changes since the container started |
| — | `docker container prune` | Remove all stopped containers |

## Image Commands

| Shorthand | Full Command | Description |
| --- | --- | --- |
| `docker images` | `docker image ls` | List local images |
| `docker pull` | `docker image pull` | Download an image from a registry |
| `docker push` | `docker image push` | Upload an image to a registry |
| `docker build` | `docker image build` | Build an image from a Dockerfile |
| `docker rmi` | `docker image rm` | Remove one or more images |
| `docker tag` | `docker image tag` | Create a tag that refers to an existing image |
| `docker history` | `docker image history` | Show the layer history of an image |
| `docker save` | `docker image save` | Export an image to a tar archive |
| `docker load` | `docker image load` | Import an image from a tar archive |
| `docker import` | `docker image import` | Create an image from a tarball (filesystem import) |
| — | `docker image inspect` | Show detailed JSON metadata for an image |
| — | `docker image prune` | Remove unused (dangling) images |

## Network Commands

Network commands were already in management format before the 1.13 restructure:

| Command | Description |
| --- | --- |
| `docker network ls` | List all networks |
| `docker network create` | Create a new network |
| `docker network rm` | Remove a network |
| `docker network inspect` | Show detailed network configuration |
| `docker network connect` | Connect a container to a network |
| `docker network disconnect` | Disconnect a container from a network |
| `docker network prune` | Remove all unused networks |

## Volume Commands

Volume commands were also already in management format:

| Command | Description |
| --- | --- |
| `docker volume ls` | List all volumes |
| `docker volume create` | Create a named volume |
| `docker volume rm` | Remove a volume |
| `docker volume inspect` | Show detailed volume metadata |
| `docker volume prune` | Remove all unused volumes |

## System Commands

| Shorthand | Full Command | Description |
| --- | --- | --- |
| `docker info` | `docker system info` | Display system-wide information (storage driver, OS, kernel, etc.) |
| `docker version` | — | Show Docker client and daemon version |
| — | `docker system prune` | Remove unused containers, images, networks, and build cache |
| — | `docker system df` | Show Docker disk usage |
| `docker events` | `docker system events` | Stream real-time events from the daemon |

---

[← Previous: Docker](22_docker.md) | [Next: Docker Images →](24_docker_image.md)
