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
| `dnf install dnf-plugins-core` | Provides `dnf config-manager`, needed to add Docker's repo. |
| `dnf config-manager --add-repo ...` | `docker-ce` is not in the CentOS repos, only Docker's own. |
| `docker compose version` | Confirms the plugin installed the space-separated `docker compose` command. |

## Notes

- `docker-compose-plugin` is the package; `docker compose` is the command it gives you.
- The task only says start, so `systemctl start docker` is enough. `enable --now` would also survive a reboot.
