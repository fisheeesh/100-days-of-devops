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

| Option | Why |
|:--|:--|
| `--driver macvlan` | Gives attached containers their own MAC address on the parent network. |
| `--subnet` / `--ip-range` | Subnet is the address space, ip-range is the pool docker allocates from. The range must sit inside the subnet. |
| `docker network inspect blog` | Shows the stored driver and IPAM config, proving all three requirements landed. |

## Notes

- The flag is `--ip-range`. `--iip-range` is rejected.
- Creating a network attaches nothing. That happens with `docker run --network blog` or `docker network connect`.
- A real macvlan on a physical LAN also needs a gateway and a parent interface. The lab gave neither, so neither is invented here.
