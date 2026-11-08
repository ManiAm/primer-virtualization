# Environment Variables in Docker

This chapter covers **environment variables** — how to pass configuration into containers at runtime, set defaults in a Dockerfile, and load variables from files.

## Default Environment Variables

Every container starts with a few built-in environment variables. Use the `env` command to inspect them:

```bash
docker run --rm alpine:3 env
```

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=4c749f3e5a22
HOME=/root
```

These are set by the base image and Docker itself. Any variables defined with `ENV` in the Dockerfile are also included here.

## Passing Variables at Runtime (`-e`)

Use the `-e` (or `--env`) flag to pass environment variables when starting a container:

```bash
docker run --rm -e COLOR=blue alpine:3 env
```

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=d1208bb03724
COLOR=blue
HOME=/root
```

You can pass multiple variables by repeating the flag:

```bash
docker run --rm -e COLOR=blue -e SIZE=large alpine:3 env
```

To pass a variable from your **host shell** without setting a value, Docker forwards the host's current value:

```bash
export API_KEY=abc123
docker run --rm -e API_KEY alpine:3 env
# API_KEY=abc123 (inherited from the host)
```

## Loading Variables from a File (`--env-file`)

For multiple variables, use an **env file** instead of repeating `-e` flags:

```bash
docker run --rm --env-file app.env alpine:3 env
```

Where `app.env` contains:

```
APP_NAME=demo
PORT=8080
```

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=9433541f240b
APP_NAME=demo
PORT=8080
HOME=/root
```

**Env file syntax rules:**

| Rule | Example |
| --- | --- |
| One variable per line | `KEY=value` |
| Comments start with `#` | `# this is a comment` |
| Blank lines are ignored | |
| No quotes needed | `APP_NAME=demo` (not `APP_NAME="demo"`) |
| Quotes become part of the value | `APP_NAME="demo"` → the value is `"demo"` (with quotes) |

You can use multiple `--env-file` flags. If the same variable appears in more than one file, the **last file wins**.

## Setting Defaults in the Dockerfile (`ENV`)

The `ENV` instruction in a Dockerfile bakes default values into the image. These are present in every container unless overridden at runtime:

```dockerfile
FROM alpine:3
ENV APP_NAME="myapp" PORT=8080
CMD ["sh", "-c", "echo APP_NAME=$APP_NAME PORT=$PORT"]
```

```bash
docker build -t env-demo .
docker run --rm env-demo
# APP_NAME=myapp PORT=8080
```

Runtime `-e` overrides the Dockerfile default:

```bash
docker run --rm -e PORT=9090 env-demo
# APP_NAME=myapp PORT=9090
```

See [Chapter 25 — ENV](25_dockerfile_instructions.md#env) for more details on the `ENV` instruction.

## Override Precedence

When the same variable is defined in multiple places, the **most specific** source wins:

| Priority | Source | Example |
| --- | --- | --- |
| 1 (highest) | `-e` flag at runtime | `docker run -e PORT=9090` |
| 2 | `--env-file` at runtime | `--env-file app.env` |
| 3 (lowest) | `ENV` in the Dockerfile | `ENV PORT=8080` |

---

[← Previous: Docker Containers](27_docker_container.md) | [Next: Container Statistics →](29_docker_stat.md)
