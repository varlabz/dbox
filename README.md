# dbox

Lightweight Docker sandbox manager for repeatable local dev shells.

`dbox` is a tiny Bash runner for creating disposable, shell-driven Docker sandboxes with project-scoped configuration. It is designed to help you:

- launch a local development container quickly
- mount the current project into the container
- inject environment variables or extra mounts
- run one-time init commands when the sandbox is created
- execute commands interactively or in batch
- clean up containers cleanly when finished

## Why this project exists

Modern local development often needs containers, but full Docker Compose stacks are heavy for simple tasks. `dbox` exists to provide:

- a minimalist workflow for ad-hoc and long-lived dev shells
- shell-native config instead of bulky YAML or Compose files
- a reproducible, repeatable way to run project-specific tooling inside Docker

## What problem it solves

`dbox` solves the friction of running Docker-based development environments while avoiding accidental or unsafe projects:

- no more manual `docker run` commands for every project
- no need to learn or maintain Docker Compose for simple shells
- reduces risk of running the wrong image in the wrong directory
- makes local container workflows fast, repeatable, and easy to document
- keeps agents and tooling isolated from the host system and from other projects

## How it works

`dbox` loads a config file (`.dbox` by default) and uses shell variables to define a sandbox environment.

The runner then:

1. determines the container name, image, and project directory
2. creates or starts the container
3. mounts the current project directory into the container
4. optionally applies extra env vars and volume mounts
5. optionally runs init commands once when the container is first created
6. executes commands inside the container, or stops/removes the container

### Config file model

Each `.dbox` or `*.dbox` file is just a Bash-style script that sets variables such as:

- `DBOX_IMAGE`
- `DBOX_CONTAINER`
- `DBOX_DIR`
- `DBOX_ENV`
- `DBOX_MOUNT`
- `DBOX_INIT`

This keeps the configuration small, readable, and editable with standard shell syntax.

## Requirements

- Docker installed and running
- Bash available to execute the `dbox` runner
- Permission to run Docker commands on your machine

## Quick Start

### 1. Use a preset

```bash
./node.dbox --start
./node.dbox --exec node --version
./node.dbox --clean
```

### 2. Use an empty `.dbox`

If you just want a default sandbox for the current directory:

```bash
touch .dbox
./dbox --start
./dbox --exec sh -lc "pwd && ls -la"
./dbox --clean
```

Defaults when `.dbox` is empty:

- `DBOX_IMAGE=alpine:latest`
- `DBOX_DIR=$(pwd)`
- `DBOX_CONTAINER` derived from the current path
- no extra env vars, mounts, or init commands

### 3. Run a one-shot command

```bash
./dbox --auto sh -lc "echo Hello from dbox"
```

## Command reference

```bash
./dbox --help
```

Available options:

- `-f, --file <file>`: load config from a file (defaults to `.dbox`)
- `-s, --start`: create or start the container
- `-e, --exec <cmd...>`: execute a command inside the container
- `-a, --auto <cmd...>`: start, exec, stop, and clean in one call
- `-i, --info`: show container details
- `-S, --stop`: stop the container
- `-c, --clean`: stop and remove the container

## Configuration reference

### Core variables

- `DBOX_IMAGE`
  - Docker image used for the sandbox.
  - Default: `alpine:latest`

- `DBOX_CONTAINER`
  - Name of the Docker container.
  - Default: derived from the current working directory and normalized for Docker-safe naming.

- `DBOX_DIR`
  - Host directory mounted into the container.
  - Default: current working directory.

### Optional variables

- `DBOX_ENV=(KEY=value ...)`
  - Extra environment variables passed to `docker run` and `docker exec`.

- `DBOX_MOUNT=("host_path:container_path" ...)`
  - Additional host volume mounts beyond the default project mount.

- `DBOX_INIT=("cmd1" "cmd2" ...)`
  - One-time init commands executed when the container is first created.
  - Commands are not re-run on subsequent starts.

## Included presets

These preset files show common sandbox configurations.

### `node.dbox`

- Base image: `node:alpine`
- Init commands: upgrade npm

Example:

```bash
./node.dbox --start
./node.dbox --exec npm --version
./node.dbox --exec node -e "console.log('hello from node')"
./node.dbox --clean
```

### `python.dbox`

- Base image: `python:alpine`
- Init commands: upgrade pip

Example:

```bash
./python.dbox --start
./python.dbox --exec python --version
./python.dbox --exec python -c "print('hello from python')"
./python.dbox --clean
```

### `shell.dbox`

- Base image: `alpine:latest`
- Init commands: install `bash`, `curl`, `jq`

Example:

```bash
./shell.dbox --start
./shell.dbox --exec sh -lc "curl --version && jq --version"
./shell.dbox --clean
```

### `pi.dbox`

- Base image: `node:alpine`
- Extra mount: `~/.pi` into `/root/.pi`
- Init commands: install `curl`, `git`, `bash`, upgrade npm, install `@mariozechner/pi-coding-agent`

Example:

```bash
./pi.dbox --start
./pi.dbox --exec pi --help
./pi.dbox --clean
```

### `dev.dbox`

A simple dev sandbox that can be customized for additional tooling and mounts.

## Image size examples

The images used by `dbox` are intentionally lightweight. For example:

- `alpine:latest` is typically under 10 MB
- `node:alpine` is often around 150 MB
- `python:alpine` is often around 120 MB

## Build a new sandbox

To create a new sandbox preset or project-specific `.dbox`:

1. Choose a base image.
2. Set `DBOX_CONTAINER` if you want a stable name.
3. Set `DBOX_DIR` only if you want a different working directory.
4. Add `DBOX_ENV` values for environment variables.
5. Add `DBOX_MOUNT` entries for extra volume mounts.
6. Add `DBOX_INIT` commands for setup steps.

Example `rust.dbox`:

```bash
#!/usr/bin/env -S dbox -f

DBOX_CONTAINER=my-rust-dev
DBOX_IMAGE=rust:alpine
DBOX_ENV=(
  RUST_LOG=info
)
DBOX_MOUNT=(
  "$HOME/.cargo:/root/.cargo"
)
DBOX_INIT=(
  "apk add --no-cache musl-dev"
)
```

Then:

```bash
chmod +x rust.dbox
./rust.dbox --auto cargo --version
```

## Typical workflows

Run a disposable command in a container and clean up automatically:

```bash
./python.dbox --auto python -m pip list
```

Open a long-lived shell for interactive work:

```bash
./node.dbox --start
./node.dbox --exec sh
# ...work interactively...
./node.dbox --stop
./node.dbox --clean
```

Inspect the sandbox state:

```bash
./node.dbox --info
```

## Behavior notes

- The project directory is always mounted into the container.
- Containers use host networking (`--net=host`).
- `DBOX_INIT` is only executed once on first creation.
- `--clean` is safe to run even if the container does not exist.
- A `.dbox` file in the current directory is the explicit trust boundary for safe execution.

## Troubleshooting

- `Config file not found`
  - Verify the config file path or create a `.dbox` in the current directory.

- `Container is already running`
  - This message is informational. Use `--exec` to run commands or `--clean` to recreate the container.

- Docker permission issues
  - Check that Docker is running and your user can execute Docker commands.
  - Verify file sharing settings for mounted directories if using Docker Desktop.
