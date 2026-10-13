# Day 42: Create a Docker Network

## Question

The Nautilus DevOps team needs Docker environments for several applications. Complete the
following network setup on `App Server 2` in `Stratos DC`:

1. Create a Docker network named `blog`.
2. Use the `macvlan` driver.
3. Configure subnet `192.168.0.0/24` and IP range `192.168.0.0/24`.

## Solution

```bash
ssh steve@stapp02

docker network ls

docker network create \
  --driver macvlan \
  --subnet 192.168.0.0/24 \
  --ip-range 192.168.0.0/24 \
  blog
# <network-id>
```

Verify the network:

```bash
docker network ls --filter name=blog

docker network inspect blog
# "Name": "blog"
# "Driver": "macvlan"
# "Subnet": "192.168.0.0/24"
# "IPRange": "192.168.0.0/24"
```

## Why

| Option or command | Why |
|:--|:--|
| `--driver macvlan` | Creates the network with the exact driver required by the task. Macvlan gives attached containers their own MAC addresses on the parent network. |
| `--subnet 192.168.0.0/24` | Defines the address space available to the Docker network. |
| `--ip-range 192.168.0.0/24` | Defines the pool from which Docker assigns container addresses. The range must belong to the configured subnet. |
| `blog` | Sets the exact network name checked by the task. |
| `docker network inspect blog` | Shows the saved driver and IPAM configuration, proving that all three requirements were applied. |

## Notes

- The correct option is `--ip-range`. The spelling `--iip-range` is invalid and causes Docker to reject the command.
- No gateway or parent interface was provided by the lab, so neither is invented here. A real
  macvlan connected to a physical LAN normally also needs environment-specific gateway and parent
  interface values.
- Creating the network does not attach any containers to it; that happens later with `docker run --network blog` or `docker network connect blog <container>`.
