# dbox

Lightweight Docker sandbox manager for repeatable local dev shells.

This project provides:

- A small Bash runner (`dbox`) that manages container lifecycle for ad-hoc dev environments.
- File-based environment presets (`*.dbox`) you can execute directly.
- A simple, shell-native configuration model (variables and arrays) instead of heavy YAML or compose files.

## What This Project Does

`dbox` creates and controls single-purpose Docker containers for development tasks. It is designed for fast, disposable workflows:

- Start a container from a chosen image.
- Mount your current project directory into the container.
- Optionally inject env vars and extra volume mounts.
- Optionally run one-time init commands (package installs, tool setup).
- Execute commands interactively.
- Stop and clean the container when done.

Important safety model:

- A local `.dbox` file in the directory is the critical safety gate.
- It proves you are intentionally running `dbox` for that directory/project.
- It avoids accidentally using default behavior from the wrong folder.
- It keeps image, mounts, env, and init commands scoped to where you run.

The included presets target:

- Node development (`node.dbox`)
- Python development (`python.dbox`)
- Minimal shell utilities (`shell.dbox`)
- PI coding agent setup (`pi.dbox`)

## Requirements

- Docker installed and running.
- Bash (the runner is a Bash script).
- Permission to run Docker commands on your machine.

Optional but recommended:

- Add the project directory to your `PATH`, or call `./dbox` directly.

## Quick Start

1. Ensure the directory you run in has a `.dbox` (or use one of the provided `*.dbox` preset files).

- This is critical for safe usage.
- Treat the presence of `.dbox` as an explicit opt-in that the directory is allowed to run inside Docker.

2. Start a container using a preset:

```bash
./node.dbox --start
```

3. Execute a command inside it:

```bash
./node.dbox --exec node --version
```

4. Stop and remove it:

```bash
./node.dbox --clean
```

## Fastest Default Setup (Empty .dbox)

If you want the simplest possible setup in a directory, create an empty `.dbox` file and use defaults.

```bash
touch .dbox
./dbox --start
./dbox --exec sh -lc "pwd && ls -la"
./dbox --clean
```

What defaults this uses:

- `DBOX_IMAGE=alpine:latest`
- `DBOX_DIR=$(pwd)`
- `DBOX_CONTAINER` generated from the current path
- No extra env vars, mounts, or init commands

This is the fastest safe path for ad-hoc usage because the directory still has an explicit `.dbox` trust marker.

Note: `.dbox` only needs execute permissions when you intend to run it directly (for example `./.dbox --start`).

## Command Reference

Run help:

```bash
./dbox --help
```

Aliases: `-h`, `--help`, and `help`.

Supported options:

- `-f, --file <file>`: load config from file (defaults to `.dbox` in current directory)
- `-s, --start`: create/start container
- `-e, --exec <cmd...>`: execute command in running container
- `-a, --auto <cmd...>`: start → exec → stop → clean in one call
- `-i, --info`: show container details
- `-S, --stop`: stop container
- `-c, --clean`: stop and remove container

## Configuration Reference

Each `*.dbox` file is a shell script that sets variables consumed by `dbox`.

### Core Variables

- `DBOX_IMAGE` (default: `alpine:latest`)
  - Docker image used for the sandbox.

- `DBOX_CONTAINER` (default: sanitized current path)
  - Container name. If omitted, generated from current working directory and normalized for Docker-safe naming.

- `DBOX_DIR` (default: current working directory)
  - Working directory mounted into the container at the same absolute path.

### Optional Variables

- `DBOX_ENV=(KEY=value ...)`
  - Extra environment variables passed to `docker run` and `docker exec`.

- `DBOX_MOUNT=("host_path:container_path" ...)`
  - Additional volume mounts besides the default project mount.

- `DBOX_INIT=("cmd1" "cmd2" ...)`
  - Init commands executed once when a container is first created.

## Included Presets

### node.dbox

- Image: `node:alpine`
- Init: upgrades npm to latest

Example:

```bash
./node.dbox --start
./node.dbox --exec npm --version
./node.dbox --exec node -e "console.log('hello from node')"
./node.dbox --clean
```

### python.dbox

- Image: `python:alpine`
- Init: upgrades pip to latest

Example:

```bash
./python.dbox --start
./python.dbox --exec python --version
./python.dbox --exec python -c "print('hello from python')"
./python.dbox --clean
```

### shell.dbox

- Image: `alpine:latest` (default)
- Init: installs `bash`, `curl`, `jq`

Example:

```bash
./shell.dbox --start
./shell.dbox --exec sh -lc "curl --version && jq --version"
./shell.dbox --clean
```

### pi.dbox

- Image: `node:alpine`
- Mounts `~/.pi` to `/root/.pi`
- Init: installs `curl`, `git`, `bash`; upgrades npm; installs `@mariozechner/pi-coding-agent`

Example:

```bash
./pi.dbox --start
./pi.dbox --exec pi --help
./pi.dbox --clean
```

## Custom Configuration Example

Create your own `.dbox` file:

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

Then run:

```bash
chmod +x .dbox
./.dbox --auto cargo --version
```

## Typical Workflows

Run a single command in a disposable environment:

```bash
./python.dbox --auto python -m pip list
```

Open a long-lived dev container session:

```bash
./node.dbox --start
./node.dbox --exec sh
# ...work interactively...
./node.dbox --stop
# can start again or clean to remove container
./node.dbox --clean
```

Inspect container details:

```bash
./node.dbox --info
```

## Notes and Behavior

- Container networking uses host mode (`--net=host`).
- The project directory is always mounted into the container.
- `DBOX_INIT` runs only when creating a new container, not every start.
- `--clean` performs stop and remove.
- `--clean` is idempotent and safe to run even if the container does not exist.
- Keeping a `.dbox` file in each project directory is strongly recommended as a protection mechanism and explicit trust boundary.

## Troubleshooting

- "Config file not found"
  - Ensure `--file` path is correct, or create a `.dbox` in current directory.

- "Container is already running"
  - This is informational; use `--exec` directly or `--clean` to recreate.

- Docker permission issues
  - Verify Docker daemon is running and your user can run Docker commands.
  - Verify permission to shared directories. Docker -> Settings -> Resources -> File sharing
