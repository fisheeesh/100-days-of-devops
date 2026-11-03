# Day 42: Create a Docker Network

## Question

The Nautilus DevOps team needs to set up several docker environments for different applications.
One of the team members has been assigned a ticket where he has been asked to create some docker
networks to be used later. Complete the task based on the following ticket description:

a. Create a docker network named as `blog` on App Server `2` in `Stratos DC`.

b. Configure it to use `macvlan` drivers.

c. Set it to use subnet `192.168.0.0/24` and iprange `192.168.0.0/24`.

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
