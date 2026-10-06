# Day 35: Install Docker Packages and Start Docker Service

## Question

The Nautilus DevOps team aims to containerize various applications following a recent meeting
with the application development team. They intend to conduct testing with the following steps:

1. Install `docker-ce` and Docker Compose packages on `App Server 2`.
2. Start the `docker` service.

## Solution

```bash
ssh steve@stapp02

sudo dnf -y install dnf-plugins-core

sudo dnf config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

sudo dnf install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

sudo systemctl start docker
```

Verify the packages and service:

```bash
docker --version
docker compose version

sudo systemctl is-active docker
# active
```

## Why

| Command | Why |
|:--|:--|
| `dnf install dnf-plugins-core` | Installs `dnf config-manager`, which is needed to add Docker's package repository. |
| `dnf config-manager --add-repo ...` | Adds Docker's official CentOS repository, where the `docker-ce` packages are published. |
| `dnf install ...` | Installs Docker Engine, its CLI and runtime, Buildx, and the Docker Compose plugin. |
| `systemctl start docker` | Starts the Docker daemon as required by the task. |
| `docker compose version` | Confirms that `docker-compose-plugin` installed the current space-separated `docker compose` command. |
| `systemctl is-active docker` | Returns `active` only when the Docker service is running. |

## Notes

- `docker-compose-plugin` is the package name; `docker compose` is the command it provides.
- The task only asks to start Docker, so `systemctl start docker` is sufficient. `enable --now` would additionally configure it to start automatically after reboot.
